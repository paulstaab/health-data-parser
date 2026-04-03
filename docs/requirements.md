# Requirements

## Purpose

This document defines the product requirements independent of implementation details.

## Requirement IDs

- IDs are stable and unique.
- Prefixes indicate domain:
  - `CLI-*`: command-line interface and argument handling
  - `PARSE-*`: data extraction and XML/GPX parsing
  - `COMP-*`: computed metrics (duration, pace, splits)
  - `OUT-*`: output formatting

## Requirements

### CLI

- `CLI-001`: The tool shall accept an optional global `--file` path pointing to an Apple Health export ZIP file; when omitted, it shall default to `./export.zip` in the current working directory.
- `CLI-002`: The tool shall provide a `running list` subcommand that prints all running workouts as a table.
- `CLI-003`: The tool shall provide a `running show <TARGET>` subcommand that prints details for a single workout. `<TARGET>` is either a global 1-based run ID or the literal `latest` (case-insensitive) to select the most recent running workout.
- `CLI-004`: Running workouts shall be ordered chronologically by `startDate` before assigning global 1-based run IDs from the full unfiltered running-workout list; `latest` shall select the final workout in that full list and display its global run ID.
- `CLI-005`: The `running list` and `running records` subcommands shall each accept an optional `--year` flag to filter workouts to a single calendar year.
- `CLI-006`: The `running list` and `running records` subcommands shall each accept optional `--from` and `--to` flags (both required together) to filter workouts to an inclusive date range.
- `CLI-007`: Within `running list` and `running records`, `--year` and `--from`/`--to` shall be mutually exclusive.
- `CLI-008`: When no workouts match the active filter for `running list` or `running records`, the tool shall print a clear message or placeholder output and exit successfully.
- `CLI-009`: `running show <INDEX>` shall exit with an error when the requested global run ID is 0 or exceeds the number of runs in the full unfiltered chronological running-workout list; `running show latest` shall never produce this error.
- `CLI-010`: The tool shall provide a `running records` subcommand that reports the longest run and the fastest qualifying 5k, 10k, half marathon, and marathon runs.
- `CLI-011`: The `running list` and `running records` subcommands shall each accept an optional `--month` flag with integer values from 1 through 12.
- `CLI-012`: `--month` shall require `--year` and shall not be accepted by `running show`.

### Parsing

- `PARSE-001`: The tool shall read `export.xml` from inside the ZIP without extracting the archive to disk.
- `PARSE-002`: The tool shall extract only workouts with `workoutActivityType="HKWorkoutActivityTypeRunning"` and ignore all other activity types.
- `PARSE-003`: The tool shall support self-closing `<Workout ... />` elements (older export format, no child elements).
- `PARSE-004`: The tool shall support `<Workout>` elements with child elements (newer Apple Watch export format).
- `PARSE-005`: When a `<WorkoutStatistics type="HKQuantityTypeIdentifierDistanceWalkingRunning">` child is present, its `sum` value shall be preferred over the `totalDistance` attribute for distance.
- `PARSE-006`: Distance values in miles shall be converted to kilometres (`× 1.60934`).
- `PARSE-007`: Energy burned shall be extracted from a `<WorkoutStatistics type="HKQuantityTypeIdentifierActiveEnergyBurned">` child element; `kJ` values are stored as-is, `kcal` values are converted to `kJ` (`× 4.184`).
- `PARSE-008`: The recording app (`sourceName`) and device (`device`) shall be extracted from `<Workout>` attributes; XML entities in these values shall be unescaped.
- `PARSE-009`: Indoor/outdoor environment and manual-entry flag shall be extracted from `<MetadataEntry>` children with keys `HKIndoorWorkout` and `HKWasUserEntered`.
- `PARSE-010`: The GPX file path shall be extracted from the `<FileReference>` child of `<WorkoutRoute>`.
- `PARSE-011`: GPX trackpoints shall be parsed from the referenced GPX file inside the ZIP, extracting latitude, longitude, and UTC timestamp from each `<trkpt>`.
- `PARSE-012`: Heart rate records shall be collected from `export.xml` by scanning for `<Record type="HKQuantityTypeIdentifierHeartRate">` elements whose time window overlaps the workout's start–end interval.

### Computed Metrics

- `COMP-001`: Duration shall be computed as the difference between `endDate` and `startDate`, formatted as `M:SS` and interpreted as minutes and seconds.
- `COMP-002`: Pace shall be computed as duration divided by distance in km, formatted as `M:SS` and interpreted as minutes per kilometre.
- `COMP-003`: Per-kilometre splits shall be computed by accumulating haversine distances between consecutive GPX trackpoints and recording elapsed time at each 1 km boundary.
- `COMP-004`: Average heart rate per split shall be computed by averaging all HR records whose time window overlaps the split's time window.
- `COMP-005`: `running records` shall choose the longest run by maximum total distance among workouts matching the active time filter.
- `COMP-006`: `running records` shall choose each fastest qualifying distance record by the lowest whole-workout average pace among runs whose total distance meets or exceeds the exact threshold for that category: 5.0 km, 10.0 km, 21.0975 km, and 42.195 km.
- `COMP-007`: When multiple workouts tie for the same `running records` category, the earliest workout in chronological order shall win.

### Output

- `OUT-001`: `running list` shall render a Markdown table with columns: `Run ID`, `Date`, `Start`, `End`, `Duration (min)`, `Distance (km)`, `Pace (min/km)`.
- `OUT-002`: `running show` shall render a key-value detail block including the global run ID, date, start, end, duration, distance, and pace; and optionally energy, source, device, environment, and manual-entry note when those fields are present in the data.
- `OUT-003`: When GPS data is available, `running show` shall render a per-km splits table below the detail block with units in the header row. Complete km splits are labelled by their km number; the final partial km is labelled with a `~` suffix and its pace is extrapolated to per-km so it is directly comparable to complete splits.
- `OUT-004`: The splits table shall include an `Avg HR (bpm)` column only when at least one HR record overlapping the workout is found.
- `OUT-005`: When no GPS data is available for a workout, `running show` shall display "No GPS data available for splits."
- `OUT-006`: `running records` shall render a Markdown table with columns: `Record Type`, `Run ID`, `Date`, `Duration (min)`, `Pace (min/km)`, `Distance (km)`.
- `OUT-007`: In all commands that display or accept a run ID, the run ID shall always be the global 1-based index from the full unfiltered chronological running-workout list, including filtered `running list` and `running records` output.
- `OUT-008`: `running records` shall always render all five record rows; when no workout qualifies for a category, the row shall contain placeholder values.
- `OUT-009`: In all output tables, units shall appear only in header cells and never be repeated in data cells.
