# Illustrative example: a bounded export

> This is a fictional walkthrough using synthetic records. It describes a delivery pattern, not a real client project or a claim about production deployment.

## Requirement

A signed-in user can export records they own. The export must never include another user's records.

## Implementation

The developer adds an export endpoint that derives the user identity from the authenticated request, filters records by that identity, and returns only the matching rows. A test creates two synthetic users and one record for each, then checks each export contains only its owner's record.

## Independent review and fix

An independent reviewer notices that a second export route still queries all records and can bypass the ownership filter. The reviewer records the finding against the candidate commit. The developer applies the same ownership check to the second route and adds a regression test proving that route also excludes the other user's record.

A fresh review of the revised candidate confirms that both routes enforce the ownership rule and that the regression test covers the reported gap. The earlier review result is retained as history; it is not treated as approval of the changed candidate.

## Simulated release evidence

In this fictional scenario, the release record identifies the exact candidate commit, the passing focused and full test commands, the independent review result, and a staging readback showing the exported rows for each synthetic user. A named human approves release only after reviewing that evidence. The deployed version is then recorded and checked against the candidate commit.

The example illustrates the sequence: requirement → implementation → independent finding → fix → re-review → evidence-bound release decision. It does not establish that any real system followed this sequence.
