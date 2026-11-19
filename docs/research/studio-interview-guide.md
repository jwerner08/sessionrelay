# Studio interview guide

A guide for interviewing a studio owner before building for them. The goal is to learn how the studio works today, so the design follows real behaviour instead of our assumptions.

## How to run it

- **60-75 minutes.** Ask the ★ questions first if time is short; they test assumptions the design depends on most.
- **Ask about the past, not the future.** "Walk me through the last booking you took" beats "would you use online booking?" People are accurate about what they did and optimistic about what they would do.
- **Don't pitch.** Describe SessionRelay only at the end, if at all. Once they know the idea, answers drift toward being polite.
- **Ask for artifacts.** A screenshot of their calendar, a sample quote, an invoice, their cancellation policy text. These are worth more than descriptions.
- **Dig with "why" and "what happened next."** When they mention a problem, ask how often it happens and what it cost them.
- **Record with permission**, and keep raw notes **outside this public repo**. Bring back only the resulting decisions (see [After the interview](#after-the-interview)).

The tag after each question names the decision (D-number in [decisions.md](../decisions.md)) or issue it informs.

## 1. The studio today

1. ★ Describe the studio: rooms, what each is used for, and who works there. *(#11, product.md)*
2. How many sessions in a typical week? Busiest days and times? *(availability, load)*
3. What's the mix of work: tracking, overdubs, mixing, mastering, audiobooks, podcasts, rehearsal, other? Roughly what share each? *(#11 session types)*
4. What are your opening hours? Do sessions ever run late or overnight? How often? *(#10 overnight Decide)*
5. Which country and currency, and do you charge sales tax or VAT on sessions? *(D30, payments.md)*

## 2. How a booking happens now

6. ★ Walk me through the last booking you took, from first contact to the session being paid for. Who did what, in which tool? *(whole booking.md)*
7. ★ Where do booking requests arrive: email, phone, text, Instagram/WhatsApp DMs, website form, walk-ins? Roughly what share each? *(#12 public page, #15 admin bookings)*
8. ★ What tools do you use to run the schedule today (Google Calendar, spreadsheet, paper, booking software)? What do you like and hate about them? *(D19, competitive context)*
9. What's the most annoying part of scheduling right now? When did it last go wrong, and what did it cost? *(priorities)*
10. Have you ever double-booked a room or an engineer? What happened? *(D5)*

## 3. Clients

11. Who are your clients (bands, solo artists, labels, narrators, podcasters, businesses)? *(#12 form fields)*
12. ★ What share are repeat clients versus new? How do repeat clients usually book? *(D13 email verification, deposit exemptions)*
13. Do clients ask for a specific engineer, or do they leave it to you? *(D9 any-engineer default)*
14. Would your clients be comfortable booking themselves on a web page, or do they expect to talk to someone first? *(core product assumption)*
15. Do clients ever ask to pick a specific room? *(#12 room choice Decide)*
16. Have you had fake, spam or no-commitment requests that blocked time? *(D13, hold limits)*

## 4. Engineers and staff

17. ★ Are engineers employees, freelancers, or a mix? Do any also work at other studios? *(D37, memberships)*
18. ★ Who decides which engineer takes a session today? Do engineers pick work themselves? *(D9, D10, #15 reassignment)*
19. Who else touches the schedule: a manager, front desk, assistant? What should they be able to do that engineers can't? *(roles, D37)*
20. ★ Do your engineers use Google Calendar for their personal life? Would they connect it? Any who wouldn't? *(D19-D22)*
21. Do engineers have fixed working hours, or do they take sessions whenever they're free? *(#11 working hours)*
22. How do engineers tell you they're away? How much notice? *(OOO, D15)*
23. Are engineers' rates set by them or by you? Do they differ by type of work? *(#11 rates)*

## 5. Rooms and sessions

24. ★ What are the rules about which work happens in which room? Any hard rules ("never track drums in C")? *(#11 capabilities)*
25. Do you need setup or teardown time between sessions? How long? *(currently not in V1)*
26. What's the shortest session you'll book? Do you sell half-days, full days, or lockouts (exclusive multi-day use)? *(#10 minimum duration, possible new booking type)*
27. ★ Is mixing or mastering scheduled as room time, or done by an engineer alone and delivered later? Is it priced per hour or per song? *(core assumption: everything is a timed room booking)*
28. Do room capacity or equipment matter when booking? *(#11 capacity/equipment Decide)*

## 6. Quoting and pricing

29. ★ How do you price a session today: fixed hourly rates, day rates, packages, per song, per project, custom every time? *(D27, quote formula in booking.md)*
30. ★ Do you send a quote before every booking, or do most clients already know the price? Would you show prices on a public page? *(V1 no-prices-before-quote rule)*
31. What adjustments are common: repeat-client discounts, long-session discounts, weekend or evening rates? *(#11 rate variants Decide)*
32. How quickly do you usually reply to a booking request? How long do clients take to say yes? *(TTL defaults: 72h request, 48h quote)*
33. How long would you hold a date for someone who hasn't confirmed? *(D7, max hold)*

## 7. Timing

34. ★ How much last-minute business do you get (same day, next day)? *(min lead time default of 72h could block it)*
35. How far ahead do people book? Any booked more than 6 months out? *(max lead time 180 days)*
36. Do you have weekly or regular clients on a fixed slot? How do those get booked and changed? *(D12 recurring)*
37. How often do sessions get moved? Who usually asks, you or the client? *(rescheduling, #15)*

## 8. Deposits, cancellations and no-shows

38. ★ Do you take deposits today? How much, when, and how? Who don't you ask for one? *(D27)*
39. ★ What's your cancellation policy, and do you actually enforce it? *(D28 refund cutoff)*
40. How often do clients not show up? What do you do when they don't? *(D17)*
41. When a session runs over, how do you handle the extra time and charge for it? *(close-out, payments.md)*
42. How and when do clients pay the balance: card on the day, bank transfer, invoice afterwards? *(D29)*

## 9. QuickBooks and accounting

43. ★ QuickBooks Online or Desktop? *(QuickBooks integration feasibility)*
44. ★ What do you use QuickBooks for: invoicing clients, taking card payments (QuickBooks Payments), or only bookkeeping? *(D26, D29)*
45. Do you have a Stripe account, or any other card processor? *(D26)*
46. Does an accountant manage the books? How should deposits be recorded? Can we talk to them? *(deposit accounting)*
47. Are your clients set up as customers in QuickBooks today? How many? *(client import)*

## 10. Communication

48. Which email address do clients write to? Is it Gmail or Google Workspace? *(D24 Gmail sender)*
49. Do clients prefer email, text, or messaging apps for confirmations and reminders? *(SMS is not in V1)*
50. Do you send reminders before sessions today? When? *(notifications.md)*

## 11. Data and trust

51. What client information do you keep, and has anyone ever asked you to delete theirs? *(D36)*
52. Is there anything about your clients or pricing you would not want in a cloud tool? *(trust, privacy)*

## 12. Wrap-up

53. ★ If you could fix one thing about how bookings work, what would it be?
54. ★ What would a new tool have to do for you to switch? What would make you stop using it within a month? *(pilot success criteria)*
55. What do you pay for your current tools? *(D3 pricing, later)*
56. Can I see your calendar, a quote and an invoice? Could I talk to one of your engineers? *(follow-up)*

## After the interview

1. Write raw notes privately within a day, while memory is fresh.
2. For each answer that confirms or contradicts a decision, update [decisions.md](../decisions.md): confirm it, or mark it superseded and add the new decision.
3. Resolve the matching "Decide:" lines in the GitHub issues, and add work items for anything new (for example lockout bookings, per-song pricing, a QuickBooks push).
4. Record the defaults that changed (lead times, TTLs, deposit percent, refund cutoff) as the seed values for this studio, not as code constants.
5. Keep identifying details (studio name, client names, prices, revenue) out of the repo.
