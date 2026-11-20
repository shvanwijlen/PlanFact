# Spec: klussen aanmaken vanuit berichten-apps (intake)

> Aanvulling op de hoofdspecificatie. Beschrijft hoe berichten uit WhatsApp, Signal, Telegram en Messenger een klus worden, en hoe de afzender als klant wordt herkend.
> Status: idee/ontwerp, nog niet gebouwd.

## Doel
Een gebruiker (vakman/bedrijf) moet een aanvraag die via een berichten-app binnenkomt met zo min mogelijk handelingen kunnen omzetten in een **klus**, gekoppeld aan de juiste **klant**.

## Kernbegrip: Binnenkomend verzoek (`IntakeItem`)
Alle bronnen leveren hetzelfde object aan een centrale **intake-inbox**. Pas daarna wordt het een klus. Hierdoor kunnen nieuwe bronnen worden toegevoegd zonder het datamodel te wijzigen.

```
IntakeItem
  id
  tenantId            // bedrijf/gebruiker van de app
  bron                // share | paste | vcard | whatsapp_api | telegram_bot | messenger_api
  ontvangenOp
  tekst               // berichttekst (kan leeg zijn)
  media[]             // foto's/bestanden
  afzender            // optioneel, zie hieronder
    telefoon?         // E.164, bv. +31612345678
    naam?
    telegramUserId?
    messengerPsid?
  extractie           // automatisch herkend uit tekst
    telefoon[]  adres?  datum?  email?
  klantId?            // na matching
  matchStatus         // geen | voorgesteld | bevestigd
  klusId?             // na omzetten
  status              // nieuw | verwerkt | genegeerd
```

Klant krijgt naast de gewone velden een lijst **kanaal-identiteiten** (telefoon, telegramUserId, messengerPsid) zodat één klant via meerdere kanalen herkend wordt.

## Bronnen, gefaseerd

### Fase 1 – zonder externe koppelingen (bouw eerst)
1. **Delen (share target)**: app registreert zich als deeldoel voor tekst, afbeeldingen en contactkaarten (vCard). Werkt vanuit alle vier de berichten-apps.
   - Android: intent-filter `ACTION_SEND` / `ACTION_SEND_MULTIPLE` voor `text/plain`, `image/*`, `text/x-vcard`.
   - iOS: Share Extension.
   - Let op: gedeelde berichten bevatten **geen** afzendergegevens. Daarom:
2. **Contactkaart delen (vCard)**: gebruiker deelt het contact van de klant (los of samen met het bericht) → naam + telefoon → matching.
3. **Plakken**: tekstveld "plak bericht" met automatische herkenning van telefoonnummer, adres, datum, e-mail.
4. (Optioneel) **WhatsApp-chatexport** (.txt/.zip) inlezen: chatnaam uit bestandsnaam, afzender per regel.

### Fase 2 – directe koppelingen (afzender automatisch bekend)
5. **Telegram-bot** (Bot API, webhook): gratis en eenvoudig; levert user-id en naam, telefoon alleen als klant die deelt.
6. **WhatsApp Business Platform (Cloud API)** via Meta:
   - Berichten lopen via Meta's servers → webhook naar onze backend. Levert telefoonnummer + profielnaam.
   - Klanten (tenants) koppelen hun nummer via **Embedded Signup** (knop "Koppel WhatsApp" → Facebook-login → nummer kiezen → sms-code; ±10 min).
   - Wij moeten eenmalig **Tech Provider** worden bij Meta (bedrijfsverificatie + app review, enkele weken). Alternatief: via een BSP (360dialog, Twilio, Bird).
   - Aandachtspunten: privé-WhatsApp-nummers kunnen niet; nummer dat in de WhatsApp Business-app staat kan mogelijk via "coexistence" (beschikbaarheid per land controleren). Antwoorden binnen 24 u op klant-initiatief zijn gratis; zelf initiëren gaat via betaalde templates. Actuele voorwaarden/prijzen bij Meta checken.
7. **Messenger** (Messenger Platform via Facebook-pagina): levert naam + page-scoped ID, geen telefoon.
8. **Signal**: geen officiële API → niet ondersteunen.

## Klant-matching
1. Exacte match op kanaal-identiteit (telefoon genormaliseerd naar E.164, telegramUserId, messengerPsid).
2. Anders: match op telefoon/e-mail uit de extractie → **voorstel**, gebruiker bevestigt.
3. Anders: "Nieuwe klant" vooringevuld met beschikbare naam/telefoon.
4. Na bevestiging de nieuwe kanaal-identiteit aan de klant toevoegen.

## Gebruikersflow
Delen/plakken/webhook → IntakeItem in inbox → app toont voorgestelde klant + herkende gegevens → gebruiker tikt **"Maak klus"** (velden vooringevuld: omschrijving = tekst, foto's als bijlage, adres/datum indien herkend) → IntakeItem status `verwerkt`, gekoppeld aan klus.

## Technische randvoorwaarden
- **Native app nodig** (bv. React Native/Expo of Flutter): een PWA kan op iOS geen deeldoel zijn. Bij Expo: config plugin / dev build voor share extension.
- Backend met publieke webhook-endpoints (fase 2), webhook-signatuur verifiëren, idempotent verwerken (berichten kunnen dubbel binnenkomen).
- Multi-tenant: elke koppeling (WhatsApp-nummer, bot) hoort bij één tenant.
- **AVG**: wij zijn verwerker voor onze klanten (verwerkersovereenkomst); Meta/Telegram zijn subverwerkers. Bewaartermijn intake-items vastleggen; genegeerde items na X dagen verwijderen.

## Buiten scope (voorlopig)
- Uitlezen van notificaties (Android NotificationListener): fragiel en privacygevoelig.
- Berichten terugsturen vanuit de app (kan later via dezelfde koppelingen).
