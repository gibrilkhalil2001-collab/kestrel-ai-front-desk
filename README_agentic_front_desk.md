# Kestrel: An Agentic AI Front Desk for Service Businesses

Multi channel conversational agent that answers enquiries on web chat, WhatsApp, voice transcripts
and email, qualifies the lead against a rubric the business agrees with, checks real availability,
books a real slot, writes it to a real CRM, and hands over to a person the moment it reaches
something it should not be deciding.

Everything runs end to end with no API key. The model layer sits behind a provider boundary with a
deterministic offline implementation, so every number below is reproducible offline and in CI.

**Notebook:** `agentic_front_desk.ipynb` (60 cells, runnable top to bottom)

---

## Why I built this

I started Baron AI Solutions to solve one problem for small service businesses, and it is always
the same problem stated differently. A dental practice, a driving instructor, a mobile mechanic, a
plumber: they all lose money on the phone. Somebody calls at seven in the evening and nobody
answers. Somebody messages on a Saturday asking whether you cover their postcode and what it costs,
and by Monday they have booked with someone else.

The owner is not doing anything wrong, they are doing the job. But the enquiry that arrives while
they are with a customer is worth the same as the one that arrives while they are at their desk,
and only one of them gets answered.

The hard part is not generating a friendly reply. Any model does that. The hard part is getting a
usable phone number out of a transcript that says "oh seven seven double oh nine double oh one two
three", never inventing an appointment slot or a price, knowing that "it is flooding through the
ceiling right now" needs a human in thirty seconds rather than a booking form, and doing all of it
without the owner ever having to check whether it made something up.

---

## What it does

- **Four channels, one pipeline.** Web chat, WhatsApp, voice transcripts and email normalise
  through adapters into one message shape. The voice adapter expands spoken digits, strips filler
  words and flags low transcription confidence. The email adapter removes signatures and quoted
  threads.
- **Slot filling with authoritative validators.** The model proposes candidate facts, deterministic
  parsers dispose. A phone number the parser cannot normalise is never stored. A postcode spelled
  aloud as "B one five, two Q R" is reconstructed correctly.
- **Grounded commitments.** Prices come from the price list, times come from the calendar, and the
  outbound guard checks every number and time in the actual reply text against what the tools
  returned. A reply that offers an unbacked slot is rewritten, and twice is a handover.
- **A calendar integration that survives failure.** Retry with exponential backoff and jitter,
  idempotency keys derived from the booking, a circuit breaker, and read after write verification
  so "I could not confirm your booking" is never said about a booking that exists.
- **A real CRM.** SQLite with foreign keys, deduplication across channels, returning caller
  recognition, and an event log. The conversation writes to it as it goes, so a caller who drops out
  after three turns still leaves a contactable lead.
- **Explainable lead scoring.** A weights dictionary the owner can argue with, computed in code from
  validated slots, never by the model. Out of area is a disqualification rather than a low score.
- **Handover by policy.** Emergencies, complaints, abuse and anything needing a survey go to a
  person, with the reason recorded.

---

## Architecture

```
            +--------+
   message  |  OPEN  |  intent, returning caller lookup
            +---+----+
                v
            +--------+   out of area ----------> CLOSED   (polite decline)
            |QUALIFY |   survey needed --------> HANDOVER (priority lead)
            +---+----+   emergency ------------> EMERGENCY (human, now)
                v        complaint or abuse ---> HANDOVER
            +--------+
            | OFFER  |   calendar down --------> HANDOVER (callback promised)
            +---+----+
                v
            +--------+
            |CONFIRM |   read back, then write to calendar and CRM
            +---+----+
                v
             BOOKED
```

Four narrow agents sit behind that: intent classification, slot extraction, a planner restricted to
a closed set of moves, and a reply writer whose output is checked before it is sent. The planner
cannot invent a new kind of turn, so the conversation cannot wander.

---

## Results

Eighteen simulated callers holding real multi turn conversations with the agent, each one a
deterministic policy that reads what the agent actually said and replies accordingly. Ten
cooperative or awkward callers, eight written specifically to break it. The calendar API runs at a
15 percent failure rate throughout, because a booking agent evaluated against a healthy API is not
evaluated.

| Metric | Base set (10) | Difficult set (8) |
|---|---|---|
| Outcome accuracy | 100% | 100% |
| Handover precision / recall | 100% / 100% | 100% / 100% |
| Slot macro F1 against ground truth | 1.00 | 1.00 |
| Bookings made, and how many were valid | 4 of 4 | 7 of 7 |
| Double bookings | 0 | 0 |
| Appointments outside working hours | 0 | 0 |
| Ungrounded commitments | 0 | 0 |
| Turns to outcome, mean | 3.3 | 5.5 |
| Cost per conversation | GBP 0.0052 | GBP 0.0092 |

The difficult set includes a caller who changes the day halfway through, one who spells their
postcode aloud on a voice call, one who asks about electrics mid booking, one who gives a phone
number with a digit missing, one with two separate jobs in one message, one who replies with nothing
but question marks, and one who tries to talk the agent into a 50 percent discount by telling it to
ignore its instructions.

### Ablations

Six variants across all eighteen callers:

| Variant | Outcome acc | Slot F1 | Booked | Invalid bookings | Wrong CRM values |
|---|---|---|---|---|---|
| baseline | 100% | 1.00 | 11 | 0 | 0 |
| no slot validation | 89% | 0.68 | 10 | 0 | **16** |
| no availability tool | 100% | 1.00 | 11 | **10** | 0 |
| no handover policy | 89% | 1.00 | 13 | 0 | 0 |
| no returning caller lookup | 100% | 0.98 | 11 | 0 | 1 |
| no price grounding | 100% | 1.00 | 11 | 0 | 0 |

**The availability ablation is the result worth reading twice.** With the calendar tool removed,
outcome accuracy stays at 100 percent. Conversations still sound confident, callers still get times,
bookings still happen. Ten of the eleven bookings are invalid and there are twenty eight clashing
pairs in the diary. Every conversational metric says the system is fine and the business would be
ringing customers all week to apologise.

That is an argument about architecture, not prompting. The model is not doing anything wrong: it was
asked to be helpful and given no way to know what was free. If a commitment can only come from a
tool result, the agent cannot invent one.

Turning off the handover policy books an appointment for the caller with water coming through their
ceiling. Turning off slot validation puts sixteen wrong values into the CRM while the conversations
still read perfectly well.

---

## What is honest about this

Perfect outcome scores come from an offline provider I wrote being tested against personas I also
wrote. The notebook says so plainly in section 12.1 rather than at the bottom.

Section 12.3 documents the run where one difficult caller failed, and the cause turned out to be a
bug in my own evaluation harness: `simulate` was building the normalised message itself instead of
routing the line through the voice adapter, so the spoken postcode expander never ran. Every number
before that fix was measuring a pipeline that does not ship. I left the account in because that is
the most common way an evaluation lies to you.

Eight known failure modes are documented in section 14, including the emergency keyword list that
will miss "the ceiling has started dripping and it is getting worse", a lead rubric that structurally
caps the highest value enquiries below routine ones, and a `two_problems` caller whose second job is
quietly dropped while the transcript still reads fine.

---

## Running it

```bash
python -m pip install numpy matplotlib jupyterlab
jupyter lab agentic_front_desk.ipynb       # run all cells, top to bottom
```

To run against a real model:

```bash
export AZURE_OPENAI_ENDPOINT="https://<resource>.openai.azure.com"
export AZURE_OPENAI_API_KEY="..."
export AZURE_OPENAI_DEPLOYMENT="gpt-4o-mini"
export AZURE_OPENAI_API_VERSION="2024-10-21"
```

---

## Built with

Python 3.10+, SQLite, numpy, matplotlib. The channel adapters, spoken digit expander, UK postcode
and phone validators, time phrase parser, circuit breaker, retry logic and lead rubric are all
written out rather than imported, so they can be read, tested and argued with. Optional backend:
Azure OpenAI.

---

## Contact

Gibril Khalil, AI Engineer, Birmingham UK.
Founder, Baron AI Solutions Ltd. Full right to work in the UK, no sponsorship required.
