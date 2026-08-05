# Agentic Procurement Workflow

> **Disclaimer:** All data in this project is synthetic, modeled on real
> manufacturing scenarios. The taskboard, mail, and ERP systems are local
> mock services. Not affiliated with or derived from any employer's systems.

## Problem

Procurement is expensive and time-consuming.  In my experience, engineers
routinely spend 4 or more hours per week on it — sending requests for quotes
(RFQ), waiting on quotes, parsing through quotes, loading items into the ERP,
creating purchase requisitions (PRQ), sending them to purchasing, waiting on
confirmation, and then posting tracking information in a project management
system.  At a $50/hr base wage (~$70/hr burdened), that is roughly $14,000 per
engineer per year, before counting the opportunity cost of an engineer not
engineering.  Multiply this across a department.

The process is simple and repetitive, but it is not easily automatable.  Quotes
come in all shapes and sizes.  Items don't line up between the RFQ and the
quotes that come back — Vendor A splits your line 3 into two lines, Vendor B
substitutes an equivalent part without saying so, and Vendor C skips it
entirely.  "Compare line 3" stops having an obvious meaning.  The same part also
goes by several names (HHCS, hex head cap screw, hex bolt), so matching a
free-text description to a catalog item number is its own problem.  Lead times
are never concrete.  Any tool that quietly guesses at these will produce wrong
answers that look right.

Procurement also cannot be fully automated, because the financial and commercial
risk is real.  Errors cost money or project timeline, and both are painful to
unwind.  For that reason this workflow uses **human gates**.  A human verifies
the agent's work before a PRQ is sent, and a human creates and sends the PO.
The agent never sends a PO.

## Approach

This workflow utilizes an AI agent to take care of all the tedious data entry
and administrative work of the procurement process, while still keeping a human
in the loop to confirm critical steps in the workflow.

1. The human operator selects a project's item list and tells the agent to begin
   procurement.

2. The agent generates RFQ emails.  The email contains quantity needed, part
   number, description, and drawings (if applicable).  It requests a quote that
   includes pricing and lead time.  The agent sends the emails to the various
   vendor representatives.

3. Upon the first vendor quote received, a configurable quote window timer
   begins.  Once all expected quotes are returned or the timer runs out, the
   agent processes the quotes.  If a quote is not able to be processed or items
   don't match up, the human operator is notified.  At this point the human
   operator will disposition the issue.  Once dispositioned, the operator can
   manually tell the agent what items correlate to price and lead time.  Once all
   items are resolved, the agent creates a report for the human operator.

4. **[HUMAN GATE]** The human operator then reviews the report and tells the
   agent which vendor's quote to use.

5. The agent loads the resolved lines into the ERP.  The item number is the key,
   so the agent writes only the item number, quantity, awarded vendor, cost, and
   lead time.  The ERP's item master supplies the rest of the record.  Nothing is
   written against a guessed item number.

6. The agent assembles the PRQ (line items, awarded vendor, pricing, lead times,
   attached quotes and drawings) and submits it to purchasing.  This write is
   guarded with an idempotency key and logged, so a retry cannot submit it twice.

7. **[HUMAN GATE]** The purchasing department representative reviews the PRQ and
   quote.  If all looks good, purchasing creates a PO, sends it to the vendor,
   and the ERP is populated with the PO number.  The agent never creates the PO
   in this workflow.

8. The agent will poll the ERP system.  Once it detects that items from the PRQ
   had a PO created, it uploads that PO number to project management software
   where the human operator can keep track of order progress.

Every agent action is logged with its inputs, outputs, and a timestamp. Write
operations carry an idempotency key, so re-running a step (a retry after a timeout,
a reconciliation loop that fires twice) never double-posts to the ERP or re-sends
an RFQ.

## Architecture

An MCP server exposes the procurement workflow as a set of tools an agent can
call.  Behind those tools sit a SQLite state layer, a quote parsing pipeline, an
item resolution step, and HTTP clients for the taskboard, mail, and ERP systems.
All three backends are local mock services.

Architecture diagram and component breakdown to follow.

## Demo

WIP

## How to Run

Requires [uv](https://docs.astral.sh/uv/) and Python 3.12 or newer.

```bash
git clone git@github.com:njsav26/procurement-agent-mcp.git
cd procurement-agent-mcp
uv sync
uv run pytest
```

## What I Would Do Next

### Known Limitations

- Quote extraction is template driven.  New vendors need new templates.
- Edge cases are flagged for human intervention rather than handled.
- Cannot handle scanned or image-only PDFs.  There is no OCR capability.

### What a Production Version Requires

- Real authentication against the real systems
- Logging and metrics
- Retry logic for failed writes to the ERP
- Access control — who can approve quotes and trigger a write

### What I Would Design Differently

- TBD after implementation
