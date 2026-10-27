---
name: ardi-mplreport
description: Create or edit ARDI Python reports built with the ARDI MatPlotLib Reporting Library (ardi.util.mplreport) — report.py files, @mplreport.ardireport, FetchHistory/AQL queries, trend charts. Use when the user asks for an ARDI report, an mplreport, or a chart of ARDI property history, so the ardi.com.au docs don't need re-fetching.
---

# ARDI MatPlotLib Reporting Library (mplreport)

A summary of the docs at https://ardi.com.au/docs/mplreport:welcome, collected in October 2026. It covers what's needed to write a report without fetching the docs again. Fetch a specific doc page only for details this file marks as unknown.

## Report layout

- Each report is its **own uniquely named folder** containing a file called **`report.py`**.
- That folder goes in the ARDI site's reports folder:
  - Windows: `C:\Program Files\Optrix\ARDI\Web\Sites\Default\Reports`
  - Linux: `/opt/ardi/web/sites/default/reports`
  - Some sites use `C:\Reports`, or an administrator may have moved it.
- In this workspace, build reports as `<ReportName>/report.py` (for example, `DailyTemperatureTrends/report.py`). The user copies them to the server.
- The report range is passed to the script by the ARDI engine. The report code does not set the range.

## JSON Description File
- Each report also has a 'info.txt' file describing the report. This is a text file containing at least 5 lines, being...
1. Report Name
2. Number of hours the report covers (eg. 24 for daily, 12 for a shift)
3. The start date & time for the report rotation in YYYY-MM-DD HH:MM:SS format.
4. The timezone name (eg. Australia/Sydney) for the report rotation.
5. The folder(s) to place the report in on the reporting UI. Can be hierarchial, eg. "Daily Reports/Temperature". If multiple folders are needed, separate them with vertical bars (|).
6. Optional: the file extension of the output file. Defaults to "pdf". Can be any format, including "mp3", "pptx", "mp4" etc, as reports do not have to be PDF files - but usually will be. If the report is not a PDF, the ARDI engine will not display it in the report viewer, but will still make it available for download. Non-PDF files should not call 'report.save()'.
- **The report's time range comes from the engine that runs it.** Data calls default to that range, so "daily" means the report is run over one day, not that the code sets a range.

## Canonical template

```python
import sys
import os
#The mplreport library is often in the base folder, so add it to the search-path
sys.path.insert(0,os.path.dirname(os.path.dirname(__file__)))
import mplreport as mplreport

@mplreport.ardireport("Report Name")
def CreateReport(report,args):

   #Set up the report page
   fig,ax = report.CreatePage(1)
   report.Title()

   #Fetch the report data
   thedata = report.FetchHistory("'Temperature' PROPERTY ALLPOINTS")

   #Check for incomplete data
   if report.FetchFailure():
      report.Failed("\n".join(report.errors))

   frame = thedata.data

   #Draw the lines
   for colname in frame.columns:
        ax.plot(frame[colname],label=colname)

   #Setup the axes
   ax.margins(x=0)
   ax.grid()
   ax.legend()
   ax.set_xlabel("Time")
   ax.set_ylabel("Temperature")
   report.TimeAxis(ax.xaxis)

   report.Save()
```

Match the docs' style: 3-space indentation, `#Comment` with no space after the `#`, and `report,args` with no space after the comma.

## Decorator and function

- `@mplreport.ardireport("<Report Name>")`: the name appears in the title block.
- The decorated function receives `(report, args)`:
  - `report` is the `mplreport` object.
  - `args` holds the report arguments. **Its contents aren't documented.**
- The function must end with `report.Save()`.

## The mplreport object (`report`)

| Method | Purpose |
|---|---|
| `CreatePage(n)` | Starts a new page with `n` sets of axes and returns `fig, ax` (like `plt.subplots`; with n>1, `ax` is presumably an array). |
| `Title()` | Adds the title block: logo, report name and date range. |
| `FetchHistory(query, start=None, end=None, samples=2000, method='interp', utc=False)` | Runs an AQL query. Details below. |
| `FetchFailure()` | Returns True if any data fetch failed. |
| `errors` | A list of error strings from data fetches. |
| `ClearFailures()` | Resets the failure flag. |
| `FailOnError(label)` | Produces a graceful failure report if a fetch failed. The docs recommend calling it immediately after `FetchHistory`, for example `report.FailOnError('Temperature')`. |
| `Failed(msg)` | Replaces the report with a simple error/placeholder message. It's also called automatically on an uncaught exception. |
| `FailedAxis(ax)` | Marks one set of axes as invalid, typically inside try/except. |
| `Clear()` | Deletes the in-progress PDF so the report can start over. |
| `Save()` | Saves the report (required). |
| `TimeAxis(ax.xaxis)` | Formats a time axis to suit the report's range: HH:MM for short ranges, day+hour for a week. |
| `HeatMapTimeAxis(...)` | Creates a date/time axis when the plot has no date index. |
| `Grid(ax)` | Applies a consistent grid. |
| `SimplifyTicks(values)` | Strips text shared by all labels, such as common parts of asset names. |
| `DateFormat(dt)` | Formats a date in a style that suits the report's range. |
| `LocalTime(dt)` / `UTCTime(dt)` | Convert between UTC and local time. |
| `GetEvents(...)` | Returns a list of ARDI events. |
| `GetAnalogueColourMap()` | Returns a Matplotlib colormap for continuous data (viridis unless custom colours are defined). |
| `GetDiscreteColourMap()` / `GetDiscreteLegend()` | Return a colormap and custom legend parts for discrete or state data. |
| `AIChannel(...)` | Outputs an array of data for AI processing (flags values as normal or unexpected). |

Related: the `KPICapture` page (https://ardi.com.au/docs/mplreport:kpicapture) captures simple KPIs from daily reports for weekly or monthly summaries.

For exact signatures not listed here, fetch `https://ardi.com.au/docs/mplreport:<methodname-lowercase>`, for example `mplreport:getevents`.

## FetchHistory details

- **`query`**: an AQL query **without** the trailing `GETHISTORY`, which is added automatically.
- **`start` and `end`**: UTC datetimes. `None` means the report's own range.
- **`samples`**: defaults to 2000 and is ignored when `method='raw'`.
- **`method`**: one of `interp`, `min`, `max` or `raw`.
- **`utc=False`**: returns times in local time.
- **Return value**: an `AQLHistResponse`, or `None` on failure.
  - `.data` is a pandas DataFrame with a time index and **one column per point (asset+property)**. Use the column names as legend labels.
  - The object also carries per-point metadata.

## AQL essentials

- `'<Property Name>' PROPERTY ALLPOINTS` returns the property's history for every asset that has it. `PROPERTY` finds the single best-matching property by name.
- **`PROPERTYEX`** is similar, but only returns EXACT matches rather than partial ones (ie. 'Temperature' PROPERTY may match with 'Temperature - Oil' and 'Temperature - Water', but PROPERTYEX would only match 'Temperature').
- References:
  - https://ardi.com.au/docs/aql:aql_reference_-_alphabetical
  - https://ardi.com.au/docs/aql:example_queries

## Dataframe Column Names

Dataframe column names are the asset name and property name seperated by a space. For example, if the asset is "Oven 1" and the property is "Temperature", the column name will be "Oven 1 Temperature". Use these names for legend labels or to access specific data series.

## Testing

Reports can only be run inside an ARDI install, where `mplreport` and a server are available. If Python is available locally, at least run `python -m py_compile <ReportName>/report.py`. Tell the user the report wasn't run against ARDI.
