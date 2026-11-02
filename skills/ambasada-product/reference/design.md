# Design for people who pay in 10 seconds

Visitors open your product on a phone, inside the Village app, between two other stands. They decide
in seconds. Build for that.

## One page, one offer

- **One page.** Everything a buyer needs fits on the landing page. Extra pages only for `/thanks` and
  maybe one result page.
- **One offer.** One thing, one price. "Your name as a Romanian folk poem, 2 TON". Not a menu of six.
  Add a second offer only after the first one sells.
- **Above the fold, on a phone (375 × 667):** what it is (one line), the price in TON, the Pay button.
  Nothing to scroll past, no signup, no cookie banner.
- **Few words.** A headline of up to 8 words, one sentence under it, the button. Cut every adjective
  you can.
- **Show the result.** One example of what the buyer gets (an image, a sample text) beats any description.

## The button

- Label it with the action and the price: **Pay 2 TON**, or **Get my poem: 2 TON**.
- Big (at least 48 px tall), full width on phones, one strong colour, nothing else in that colour.
- If the buyer must type something first (a name, a photo), put that one field right above the button.
- After paying, land on `/thanks` with the result, or "Working on it…" and the result appearing when ready.
  Never a blank page.

## Look

- Plain CSS is enough. One font (the system font is fine), two colours plus grey, generous spacing.
- Large text: 18 px body on phones, a headline twice that.
- Dark text on a light background unless your idea needs otherwise.
- Works with one thumb: tap targets 48 px, no hover-only menus.
- Loads fast: no video backgrounds, no 3 MB hero images, no animation libraries.

## Words

- Say what they get, not how it works. "A logo for your stand in 1 minute", not "AI-powered logo generation".
- Prices are TON, written as `2 TON`. TON is the Village play currency; never call it crypto.
- Write in the language of your buyers. At Hack A Ton that is often Romanian and English; one is fine.

## Before and after

```
Bad:  Welcome to PoemForge™, the next-generation AI-powered creative writing platform
      [Features] [Pricing] [About] [Sign up]   …scroll…   [Get started]

Good: Your name as a folk poem
      Type a name. Get a 4-line Romanian folk poem in 10 seconds.
      [ Name: _________ ]
      [   Get my poem: 2 TON   ]
      "Ana, floare de la munte, / …"  ← one real example
```
