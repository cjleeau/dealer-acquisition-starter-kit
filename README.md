# Dealer Acquisition Starter Kit

A free, open starter kit for small car dealerships that want a simple way to buy cars from private sellers. It contains a set of screen designs and a detailed build script. Hand both to an AI coding tool ([Claude Code](https://docs.claude.com/en/docs/claude-code/overview), [Cursor](https://cursor.com), [Lovable](https://lovable.dev), [Bolt](https://bolt.new), [v0](https://v0.dev) or similar) and it has everything it needs to build a working first version.

This kit is part of the LinkedIn series *Dealership AI, one feature at a time* by [Chris Lee](https://www.linkedin.com/in/christopherjustinlee). It is not a product and there is nothing to buy.

---

## What the finished app does

1. A private seller fills in a short form on their phone: number plate, a few questions and six guided photos. There is no chatbot.
2. The app looks up the car, checks the photos for the plate, the odometer reading and possible damage, and gathers a valuation.
3. A buyer at the dealership reviews everything on one screen and sets a price range. A person sets every price.
4. The seller gets the range by SMS and books a time to bring the car in.
5. At the inspection, staff complete the checks (including PPSR), agree the final price, and both people sign on a tablet.
6. A deal pack is produced, and every step is recorded in an activity log.

Reminders, automatic follow-ups that stop as soon as the seller replies, opt-out handling and automatic deletion of old seller data are all included.

---

## What is in this repo

| File | What it is |
|---|---|
| `README.md` | This file |
| `acquisition-mvp-build-script.md` | The build script. A precise specification for an AI coding tool: data model, screens, exact wording, statuses, SMS templates, audit log, privacy rules, acceptance criteria and build order |
| `design/acquisition-flow-mockups.html` | The screen designs: five seller screens on a phone, the buyer's desk, and the offer and sign screen on a tablet. Open it in any browser |

---

## What this repo does and does not do

This repo is a blueprint, not the app itself. Nothing here runs on its own.

- **It does not create your app, a GitHub repository for your app, or a live website.** You (or your AI coding tool) create a new project, and the tool writes the code into it.
- **Where the code ends up depends on the tool you use.** Claude Code and Cursor build the app on your own computer, in a folder you choose, which you can then push to your own GitHub repository. Browser tools such as Lovable and Bolt build it in their own environment and can usually sync it to your GitHub account.
- **Putting it online is a separate step.** The build script assumes [Vercel](https://vercel.com) for hosting, [Supabase](https://supabase.com) for the database and file storage, and [Twilio](https://www.twilio.com) for SMS, and lists the environment variables each needs. It does not walk you through creating those accounts. Your AI coding tool can guide you through that when you get there.

---

## How to use it

1. **Read the build script first.** It takes about 20 minutes and tells you exactly what will be built, and what will not.
2. **Create an empty project folder** on your computer, or a new project in your browser-based tool.
3. **Give the AI coding tool both files.** Put `acquisition-mvp-build-script.md` and the mockups file into the project, then ask:

   > Read `acquisition-mvp-build-script.md` and open the mockups in `design/`. Follow section 0 of the script exactly. Start with phase 1 of the build order in section 15, and stop when phase 1 is finished so I can review it.

4. **Build one phase at a time.** Check each phase against the acceptance criteria in section 14 before asking for the next one. Do not let the tool skip ahead.
5. **Test with the sample data.** The script includes a seed command that loads fake sellers and fake cars. While sample data is loaded, SMS messages are logged but never sent.
6. **Work through the go-live checklist** in section 12.4 of the script before any real seller uses it.

---

## What you will need

**To build and test it:**
- An AI coding tool
- Some comfort following technical steps. You do not need to be a developer, but someone who is handy with software will find it much easier.

**To run it for real:**
- A [GitHub](https://github.com) account (to keep your code)
- A [Supabase](https://supabase.com) account (database, file storage and staff logins)
- A [Vercel](https://vercel.com) account or similar (hosting)
- A [Twilio](https://www.twilio.com) account and a sending number (SMS)

**Optional, the app works without them:**
- A vehicle data provider API to look up cars from the number plate (for example [AutoGrab](https://autograb.com.au)). Without it, the seller types in the make and model.
- A valuation API. Without it, staff enter the value by hand.
- A vision-capable AI model for the photo check. Without it, staff review the photos themselves.
- A PPSR provider API. Without it, staff run the search on the government [PPSR website](https://www.ppsr.gov.au) and upload the certificate.
- [MotoIQ](https://motoiq.ai) for an EV value signal on electric cars.

Every one of these has a manual fallback, so you can start with none of them.

---

## Before you use it with real sellers

Please read this part.

- **This is a starting point, not legal advice.** Contract terms, record-keeping periods and licensing rules for dealers vary by state. The build script will not let the app complete a purchase until you add your own contract wording, and that wording should come from your lawyer or industry association.
- **Seller details are personal information.** Names, mobile numbers and photos showing number plates are covered by the Privacy Act. The OAIC's [guidance on privacy and commercially available AI products](https://www.oaic.gov.au/privacy/privacy-guidance-for-organisations-and-government-agencies/guidance-on-privacy-and-the-use-of-commercially-available-ai-products) is a good place to start. Update your privacy policy, keep the consent checkbox, and only use an AI photo check on a business plan that does not train on your data.
- **SMS rules apply.** The [Spam Act](https://www.acma.gov.au/avoid-sending-spam) covers marketing messages. The app only messages sellers who have ticked the consent box, identifies the dealership in every message and honours STOP. Do not remove these.
- **A person makes every decision that matters.** The app never sends a price on its own, and the AI photo check only flags things for a person to look at. Keep it that way.
- **Test thoroughly.** AI coding tools can make mistakes. Run every acceptance test in section 14 before going live.

---

## Questions and improvements

If you build this, improve it or find a problem, please open an issue in this repo or get in touch with [Chris Lee on LinkedIn](https://www.linkedin.com/in/christopherjustinlee). Better ways of doing it are very welcome.

---

## About

Created by [Chris Lee](https://www.linkedin.com/in/christopherjustinlee) ([mrleeco.com](https://mrleeco.com)) as part of the *Dealership AI, one feature at a time* series. MotoIQ ([motoiq.ai](https://motoiq.ai)) focuses on EV value intelligence for Australia.

Brands and services named in this kit (AutoGrab, Supabase, Vercel, Twilio and others) are examples only. This kit is not affiliated with or endorsed by any of them.

## Licence

Released under the MIT Licence. See `LICENSE`. You are free to use, change and share this kit, including in commercial projects, with no warranty of any kind.
