---
concept: output
title: Output (What the Live Week Came To)
kind: derived
branch: 6-output
aka: [score, scores, SchedAlign, coverage, fill, filled, unfilled, open seats,
      overtime, OT, preference score, badge score, availability score,
      "how did we do", "how does the week look", "why did he get", "why didn't she get",
      hours, "hours short", headroom, "what can we still reach", "what would it cost",
      budget, "over budget", "under budget", committed, "minimum remaining"]

scope_note: |
  THE DERIVED LAYER over the LIVE week, BEFORE the week is worked: what the
  live schedule scored, what it costs against budget, and what it could still
  reach. Two grains — the WEEK (client, or a dispatch location) and the
  DRIVER-WEEK. Everything here is computed from [[labor]] (who wants what)
  meeting [[structure]] (what is demanded) at the seat, inside one
  [[coordinate-frame]] week. ONLY THE LIVE WEEK IS INTERPRETABLE. Candidate
  schedules a manager is still working on are not in scope until they are made
  live, and the model never reads them. What the day then actually did —
  callouts, drops, punches — is [[preparation-execution]], not this. The engine
  that computes the scores is NEVER described here: this concept holds its
  verdicts as evidence rows, not its method.

disclosure:
  citable:  [every column on the two metric tables, hours demanded / filled / unfilled,
             regular and overtime hours per driver, the band word a score falls in,
             budget / committed / minimum remaining / over-or-under, Headroom's three
             bands and the filled pointer, unfillable seats and hours]
  internal: [WeightCoverage / WeightOvertime / WeightPreference, tblEngineObjectiveProfile,
             every manifest recordset, the path matrix, threshold floor values, everything
             on tblEngineSettings the manager did not set on a screen, and every
             scenario-side table (tblScheduleScenario, tblScenarioMetricBlob,
             tblEngineRunAssignment, tblEngineRunMandate) — out of scope, never read]

grounding:
  week_grain:
    tblWeeklyScheduleMetrics: { keys: [ClientId, WeekId],
                                carries: [TotalHoursfilled, TotalHoursUnfilled, ShiftFilledPercent, AvgHoursPerLMDP,
                                          TotalOTHours, OvertimePercentage, OverallOTScore,
                                          AvailabilityPreferenceScore, BadgePreferenceScore, OverallPreferenceScore,
                                          SchedAlignScore, TotalCostFilled, TotalCostUnfilled],
                                role: the LIVE week's row — ONE per client-week, rewritten on every assignment write }
  driver_grain:
    tblLMDPWeeklyMetrics:     { keys: [ClientId, WeekId, LMDPID],
                                carries: [TotalHours, AvailabilityPreferenceScore, BadgePreferenceScore, OverallPreferenceScore,
                                          ConsecutiveDays, ConsecutiveHours],
                                role: one row per driver per LIVE week }
  the_schedule_scored:
    tblAssignmentHours:       { keys: [ClientID, WeekID, MissionSeatID, LMDPID], carries: [RegularHours, OTHours],
                                role: the LIVE split of every assigned seat's hours into regular and overtime }
  headroom:
    Dash_Headroom_Hydrated:   { params: [@ClientID, @WeekID, @DispatchLocationID, @ScenarioID (NULL = live), five engine-result arrays],
                                recordsets: { RS1: header — hours, filledHours, budget, committed, minimumLeftToFill, total, delta, unfillableSeats, unfillableHours,
                                              RS2: demand per day per seat, RS3: supply per driver (worked vs wanted),
                                              RS4: the ring — every seat in one of three bands, RS5: the two poles },
                                role: reads the LIVE schedule; says what the week could still reach and what it costs from here; dispatch-location grain }
  bands:
    tblMetricThresholds:      { keys: [MetricKey], carries: [BlueFloor, GreenFloor, YellowFloor, OrangeFloor],
                                role: turns a score into a band word. Four keys — Coverage, Overtime, Preference, SchedAlign — all at 95 / 90 / 80 / 70 today; below the orange floor is red }
  settings:   # INTERNAL. The engine's dials. Named so the model knows they exist and must not cite them.
    fn_ResolveEngineSettings: { params: [@ClientId, @ScenarioId], role: the ONLY door to tblEngineSettings; never returns NULL }
    tblEngineObjectiveProfile: { keys: [ObjectiveProfile], carries: [WeightCoverage, WeightOvertime, WeightPreference], role: the two Headroom pole weightings }
  accessors:
    panel:    Dash_Assignment_Main_Hydrated(@ClientId,@SelectedWeekId,@ScenarioID)   # RS12 = the score panel; live when @ScenarioID is NULL
    headroom: Dash_Headroom_Hydrated                                                # above
  write_doors:
    Proc_Assignment_ApplyDerived:    THE door for derived state — metrics AND the hours split together, once per batch, after ANY assignment write. Never re-implemented inline.
    Proc_SubmitAssignment_ClearWeek: empties a week and removes its metric rows
    Proc_Master_Manifest:            what the engine is HANDED — eight recordsets. INTERNAL; named so it is never cited.

scores:   # what each number is. The formulas are the engine's; these are the meanings.
  coverage:                    "hours filled ÷ hours demanded, as a percent (ShiftFilledPercent)"
  overallOTScore:              "how little of the filled work needed overtime; 100 = none"
  availabilityPreferenceScore: "a BLEND over the availability axes — day, location, overtime, standby, weekly hours"
  badgePreferenceScore:        "a BLEND over the badge axes — Transport, Role, Service"
  overallPreferenceScore:      "the roll-up of the two blends"
  schedAlignScore:             "the headline: coverage, overtime and preference blended under the client's standing weights (the weights are internal)"
  rule: "the axes inside a blend are NOT stored anywhere. No table says which axis cost a driver points."

money:   # Dash_Headroom_Hydrated RS1, all live-priced
  budget:            "every demanded hour × its seat rate"
  committed:         "what the filled seats cost as scheduled, regular plus overtime premium"
  minimumLeftToFill: "the least it would cost to fill what is still open"
  over_under:        "delta = committed + minimumLeftToFill − budget. Positive is over budget. Negative is under, and it can only be negative when seats are left empty."

relationships:
  - Output is DERIVED-FROM [[labor]] MEETING [[structure]] at the seat, inside one [[coordinate-frame]] week
  - Output is SCORED-BY the engine; the method is WITHHELD — cite evidence, never method
  - Output is RE-COMPUTED once per batch on every assignment write; the row is the CURRENT live schedule, not a history
  - Only the LIVE week is interpretable; a candidate schedule becomes readable when it is made live
  - Headroom's ring is TERRAIN (what the week could reach from empty, under relaxed settings); the pointer is the LIVE fill; the poles are the cost from here
  - A "why" about a score is ANSWERED-FROM [[labor]] preference rows and [[structure]] demand rows set beside what landed — never from a sub-score
  - What the day then did is [[preparation-execution]]

fill_reality:   # client 7293, verified 2026-09-15
  week_metric_rows: 15            # 15 live weeks carry a row
  driver_metric_rows: 1741        # over those 15 weeks
  assignment_hours_rows: 990
  threshold_keys: 4

cite: the tblWeeklyScheduleMetrics or tblLMDPWeeklyMetrics row behind any score; the tblAssignmentHours rows behind any hours; the Dash_Headroom_Hydrated recordset row behind any headroom or money claim
intents: []     # first two arrive with the schedule-feedback handover: the driver-week account (push) and the week's day tension (push)
---

## Meaning

**THE QUESTION.** Output answers: what did the live week come to. How much of
the demand is covered, at what cost in overtime and against budget, how well
the people got what they asked for, and what the week could still reach. It is
measured on the plan, before anyone drives. Once the day is worked, the
plan-versus-actual story belongs to [[preparation-execution]].

**ONLY THE LIVE WEEK.** A manager may be building other versions of a week on
the side. None of those is interpretable until it is made live, and the model
never reads them. Every number in this concept is the live schedule's number.

**TWO GRAINS, ONE SOURCE.** The week has a row — hours filled and unfilled,
coverage, overtime, the preference scores, the headline SchedAlign — and every
driver has a row for that week — their hours, their preference scores, their
longest run of days. Both are computed from the same schedule at the same
moment, so they always agree with each other.

**★ THE SCORES ARE BLENDS, AND THE AXES ARE NOT STORED.** A driver's
availability score blends day, location, overtime, standby and weekly hours.
Their badge score blends Transport, Role and Service. Nothing in the database
says which axis cost them points. So a question like *"why is her score 62"*
cannot be answered from a sub-score, because there is none. It is answered from
evidence: the preferences she set ([[labor]]), the demand on the days she
opened ([[structure]]), and the seats she actually got. Set those side by side
and the reason is visible without the engine saying a word. **This is the rule
that keeps the portal from inventing a rationale.**

**THE ROW IS NOW, NOT HISTORY.** Every assignment write — a manager placing a
seat, an engine run, a removal, an emptied week — recomputes the week's metrics
once per batch, through one door. The row you read is the schedule as it stands
this second. There is no trail of what the score used to be; if a question
needs that, it is a [[ledger]] question about the assignments, not a metrics
question.

**HEADROOM IS TERRAIN, THE POINTER IS THE SCHEDULE.** Headroom sorts every
seat in the week into three bands: fillable under the client's own settings,
fillable only under relaxed settings (an override or overtime), or unfillable.
Those bands are a property of the week's supply against its demand — they do
not move when a manager places a seat. The pointer, how much is filled, does.

**UNFILLABLE.** Unfillable hours are the hours with no eligible driver
remaining once the constraints on consecutive days and total hours are
applied, even at the relaxed settings. Nobody who could take the seat is left.

**THE MONEY.** Budget is every demanded hour at its rate. Committed is what the
filled seats cost as scheduled. Minimum remaining is the least it would cost to
fill what is still open. Over or under budget is committed plus minimum
remaining, minus budget. Positive is over. Negative is under, and it can only
be negative when seats are left empty, because budget already pays for every
demanded hour.

**NEVER THE METHOD.** The weights, the pole profiles, the manifest the engine is
handed and the matrix it solves over are named in this concept so the model
knows they exist and knows they are off limits. A manager may be told any
number the engine produced and the evidence behind it. A manager is never told
how the engine got there. *"Why did the scheduler choose X"* is refused by
design.

**DISCLOSURE.** Every score, every hours figure, every band word and every
money figure is the client's own operating picture and citable to their
manager. The engine's dials are internal.
