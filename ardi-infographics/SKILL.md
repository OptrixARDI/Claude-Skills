---
name: ARDI Display Creation
description: Documentation that covers the creation of ARDI interactive dashboards/displays in Javascript.  Use when the user asks for an re-usable display, infographic or 'ARDI display' that can be published.
---

# ARDI Infographic Reference

A summary of what you need to write an ARDI Infographic (a JavaScript display that runs on an ARDI server), so you don't have to download the docs and libraries again.

Compiled 2026-10-26 from the ARDI docs (`https://ardi.com.au/docs/`) and the library sources served by the public Optrix demo server. The library code is on the server and can change, so if something here doesn't match what you see, check the source (section 11).

---

## 1. What an Infographic is

- It's a folder in the server's *displays* directory, containing a `generate.js` and an `info.txt` file.
- `generate.js` defines `class ActiveReport extends SVGReport { ... }`. ARDI puts this class inside its own HTML page, which already includes d3, jQuery and moment.
- You choose the data points with an **AQL** (ARDI Query Language) query. The data is **live** (streamed current values), **historical** (values over a time range), or **hybrid** (history first, then live).

### Deployment

| Platform | Default displays folder |
|---|---|
| Windows | `c:\Program Files (x86)\Optrix\ardi\web\sites\default\displays` |
| Linux | `/opt/ardi/web/sites/default/displays` |

- Create one folder per Infographic. Avoid spaces in the folder name, e.g. `oven_temperature_trend/`.
- To use a different location, put a file called `displays.redirect` in the site folder. Its content is the full path of the folder to use instead.
- `info.json` (lowercase) has several values:
  'title' - The display name, e.g. `Oven Temperature Trend`  
  'folder' - The logical folder(s) the display appears in via the UI. Separate several with `|`. Leave blank for none.
  'lightmode' - Optional. If true, this visualisation can handle light or dark mode. The body of the page will have the class 'lightmode' in light mode.
  'title' - Optional. If true, a title is already present on the display and the default title in the top-left corner won't be shown.
  'background' - Optional. A fixed background colour.

---

## 2. Page environment

On the demo server the page loads these scripts, in this order:

```
/plugins/jquery/jquery.min.js
/addons/displaylist/d3.min.js          <- d3 v7.8.2
/addons/displaylist/reporting.js       <- Report / SVGReport / CanvasReport / BlankReport
/plugins/optrix/hmitt.js               <- HMITTPanel (live connection + time travel)
/addons/dexplore/moment.min.js
/plugins/optrix/timecontrols.js
/s/<site>/displaylist/converters.js.php  <- MapColour[propid], MapValue[propid] lookup functions
```

The docs also say **three.js** is available for 3D. To load other libraries, call `this.addLibrary(url)` before `initialise` runs (for example in the constructor or a field initialiser). The page waits for those libraries to load before it calls `initialise()`.

### Bootstrap sequence (from the page)

```js
report = new ActiveReport("<host>/s/<site>", "<Display Name>", defaultRange, 'https');
report.baseurl = "/s/<site>/displaylist/";
report.setSize(window.innerWidth, window.innerHeight);
report.embedded = true;            // when ?embed=true
report.initialise();
report.create();
addEventListener("resize", ...) -> report.setSize(w,h); report.resize();   // base resize() does nothing
```

`defaultRange` is either a number of minutes or `"YYYY-MM-DD HH:mm:ss to YYYY-MM-DD HH:mm:ss"`.

### Styling gotchas

- The page background is **black** by default, unless light mode is available. (`<body style="background-color: black">`). Design for dark mode or include CSS styling for both light and dark mode.
- `online.css` contains **`svg line { stroke: white; }`** and **`.axis text { fill: white; }`**. CSS overrides SVG presentation attributes, so `.attr("stroke", ...)` on a `<line>` has no effect. Use **`.style("stroke", ...)`**, and don't give elements the class `axis`.
- d3 axes draw tick lines as `<line>`, so they turn white too. After every `.call(axis)`, restyle them with `.style("stroke", ...)`.
- `SVGReport.initialise()` appends `<svg id="reportsvg" style="display:none">` with `viewBox = 0 0 width height`. The SVG stays **hidden** until `liveReport` or `query` fades it in, so `getComputedTextLength()` and `getBBox()` return 0 on the first draw. Measure text with a canvas `measureText` instead.
- The tooltip `#internal_tooltip.tooltip.simple` has a white background, black text and `max-width: 300px`. Your tooltip HTML should use dark text in dark mode.
- **Use `\u` escapes for non-ASCII characters** in JS strings (e.g. `"\u00b7"`, `"\u2026"`, `"\u00b0"`). In testing, a literal `·` came out as `Â·` because the script was read with the wrong encoding. **Check the bytes on disk afterwards** (`LC_ALL=C grep -n '[^[:print:][:space:]]' generate.js`): file-editing tools and shell quoting can quietly turn `·` back into the literal character, or drop the backslash.

---

## 3. Skeleton

```js
class ActiveReport extends SVGReport {
	initialise() {
		super.initialise();      // creates #reportsvg, sets this.svgbase (d3 selection)
		this.first = true;
	};

	create() {                   // send the query
		this.liveReport("'Temperature - Air' PROPERTY ALLPOINTS");
	}

	update() {                   // live: called each time new values arrive
		this.draw(this.livedata);
	}

	draw(data) {                 // called automatically once the first query returns
		if (this.first == true) { /* build axes etc. once */ this.first = false; }
		/* d3 join to update marks */
	}
}
```

- **initialise**: runs once. Set up variables only.
- **create**: runs once to send the AQL query. The base `update()` calls `create()` again, so override `update()` for live displays.
- **update**: live only. Called on each update cycle.
- **draw(data)**: live displays get an array of current values in column order. Historical displays get rows of `[Date, v1, v2, ...]`.

Most of the official examples draw everything once (`$('#reportsvg').html("")` on the first draw, guarded by `this.first`) and then update marks with d3 `join(enter, update)` and `transition()`.

---

## 4. `Report` / `SVGReport` API (reporting.js)

### Properties

| Property | Meaning |
|---|---|
| `this.svgbase` | d3 selection of `#reportsvg` |
| `this.sizing` | `[width, height]`; also `this.width`, `this.height` |
| `this.margin` | `{left:30, top:15, bottom:30, right:15}` by default. `drawHeader()` adds 60 to `top`. Treat `top` as the space reserved for page chrome. |
| `this.columns` | Column info in value order (see section 5 for its shape) |
| `this.livedata` | Array of current values (already parsed to numbers where possible) |
| `this.data` | Historical rows (`[Date, ...values]`) |
| `this.connection` | `HMITTPanel` (live/hybrid only) |
| `this.reportmode` | `"live"` or `"history"` |
| `this.span` | Range in seconds |
| `this.rangeformat` | String inserted where `%%` appears in a query |
| `this.updatetiming` | ms between the last two live cycles |

**Name collisions.** These names are already used by the base class. **Don't assign them** as your own fields or methods: `start`, `end` (Dates), `query`, `data`, `columns`, `name`, `server`, `width`, `height`, `span`, `options`, `connection`, `libraries`, `schema`, `margin`, `sizing`, `livedata`, `stepped`, `events`, `parseDate`. For example, defining a `start()` method fails with "this.start is not a function" because the constructor assigns `this.start = new Date()`. The examples use `this.x`, `this.y`, `this.svg` and `this.first` freely; those are safe.

### Query methods

| Method | What it does |
|---|---|
| `liveReport(aql, dots=false)` | POSTs to `/aql/api/query`, builds `columns`, subscribes each point's `code`, calls `draw(livedata)`, connects, then calls `update()` on each cycle. |
| `singleQueryReport(aql)` | Historical. Replaces `%%` with `"range": "N minutes", "grain": -<width/2>` (or start/end), POSTs to `/aql/api/table` with `metadata=true`, then `draw(rows)`. Also calls `dynamicRange()`, which replaces `#range` with a time selector. |
| `hybridReport(aql)` | Historical first, then live. Its own timer pushes averaged live samples onto `this.data` and calls `draw(this.data)`. Columns here are only `{name, sequence}`. **The page reloads when the tab becomes visible again.** |
| `aqlQuery(aql, cb)` / `rawQuery(aql, cb)` | Raw `/aql/api/query` calls (aqlQuery substitutes `%%`) |
| `query(aql, cb)` | Raw `/aql/api/table` call with `%%` substitution |
| `getEvents(cb)` | `{%%} GETEVENTS`, fills `this.events` |
| `setTimeSpan(min)`, `setSamplesPerHour(n)`, `setSteppedProperties([...])`, `setSampleSize(ms)`, `setCustomParser(fn)` | Query tuning |
| `getProxyData(name, args, cb)` | POST to `/displaylist/proxy` |
| `changeReport(name, extra, options)`, `rename(title)`, `exploreAsset(name)` | Navigation |

### Drawing helpers

- `getSequenceColour(i)`: cycles through `gold, blue, green, yellow, purple, grey, darkgreen, pink, brown, slateblue, violet, orange`. It isn't colour-blind safe; use your own palette for serious work.
- `drawAxisSet(x, y, xname, yname, xpos, ypos, width, height, name="main", entry, exit)`: draws a background rect, axis titles, a dashed `.axismarker` line, x/y axes and a grid. It uses `this.margin`, adds `<clipPath>` with a rect `#clipcontent`, and its background fill is `"#00000"` (an invalid colour).
- `drawHeader(name, sd, ed)`: draws a title and date range, and adds 60 to `margin.top`.
- `getMouseAxisPos(x, y, clientX, clientY)`: returns `[xValue, yValue]` in data space. `d3.pointer(ev, svgnode)` also works and handles the viewBox scaling.
- `formatDuration(seconds)`. `formatDate(d, from, to)` has a bug: it references an undefined `md` for spans longer than a day.
- `MapColour[PROPID](value)` and `MapValue[PROPID](value)` (from converters.js.php) give the ARDI-configured colour or text for a value.

### Tooltips

- Register a mark with `.call(this.tip)`. This binds `mouseover`, `mousemove` and `mouseleave` (no namespace) and calls the global `report.updateTooltip(ev)`. Add your own pointer handlers under a namespace (e.g. `pointermove.mine`) so the two don't overwrite each other.
- The default `updateTooltip(ev)` reads the target's `name`, `value` and `units` attributes. Override it to return an HTML string, or `null` to leave the tooltip unchanged.
- While a tooltip is open, each live cycle calls `report.updateTooltip({target})` **without `clientX`/`clientY`**. Keep your hover state in fields rather than relying on the event.
- `ShowTooltip` sets `opacity 0.75` on every `.element` and `1` on the hovered one, so elements with class `element` dim on hover.

---

## 5. Data shapes

### Live columns (`liveReport`)

Each entry is a pointlist hit from `/aql/api/query`, with `name` rewritten. A real example:

```json
{"assetid":76,"sourceid":76,"value":140.11,"rawvalue":140.11,"propid":34,
 "fullvalue":"0| Deg C|0|400|1","type":"MEASUREMENT",
 "name":"Prime Oven Zone 1 Temperature - Oven",   // rewritten: asset + " " + propname
 "assetname":"Prime Oven Zone 1",                  // added by liveReport
 "code":"76:34:measurement","units":" Deg C","min":"0","max":"400","places":"1",
 "propname":"Temperature - Oven","ern":"OVN01-1"}
```

- `units` has a leading space. `min`, `max` and `places` are **strings**.
- Order is stable (it follows the query result), so you can assign colours by column index.
- Non-measurement types: `STATUS`, `ENUM`, text. Subscription node names are `state`, `value`, `measurement` and `text`.

### History (`singleQueryReport` / `/aql/api/table`)

- The raw response is `{columns:[...], data:[[timeString, v1, v2...]...], metadata:[{assetname, propertyname, ...}]}`.
- After `simpleCleanup`, each row is `[Date (UTC-parsed), v1, v2, ...]`. A value of `"^"` becomes `null`, and non-numeric values stay strings. The columns become the `metadata` objects, with `name = assetname + " " + propertyname`.

---

## 6. Live connection & time travel (hmitt.js `HMITTPanel`)

- The connection polls the server about once a second. **`UpdateCycleComplete` (and so `update()`) only fires when at least one value changed.** If values are steady, nothing fires. For trends, sample on your own `setInterval`.
- `OnlyWhenFocused()`: the connection **closes while the tab is hidden** and reconnects when it's visible. Expect gaps in your data, and don't record stale values while `document.visibilityState == "hidden"`.
- In `liveReport`, `ConnectTo()` is called *before* `UpdateCycleComplete` is assigned. This only matters if you mock the connection.
- Live displays get **time-travel** controls automatically. The useful fields are:
  - `connection.history`: `true` in time-travel mode, `false` when live (set by `LivePlayback()`).
  - `connection.currenttime`: a moment (UTC) for the playback instant. Use `.valueOf()` for epoch ms.
  - `connection.historymode`: `1` while playing, `0` while paused.
  - `connection.playbackspeed`: seconds of data per 1-second playback tick.
  - `connection.ontimeupdate(localMoment)`: set by `liveReport` to update the time label.
  - Playback calls `UpdateCycleComplete` on each tick (once per real second).
- The current data time is therefore `c.history ? c.currenttime.valueOf() : Date.now()`. If you buffer data, reset the buffer when that time jumps backwards or by more than your window.

---

## 7. AQL quick reference

| Query | Meaning |
|---|---|
| `'Temperature - Air' PROPERTY ALLPOINTS` | Every point of that property, across all assets (live) |
| `'Temperature - Air' ALLPOINTS` | Shorter form, used in some examples |
| `'Asset' ASSET 'Prop' PROPERTY VALUES {"grain": -2000,%%} GETHISTORY` | History for one asset and property |
| `'Prop' ALLPOINTS {%%} GETHISTORY` | History for all points of a property |
| `('code1','code2') CODEPOINTS {"start": "...", "end": "...", "grain": -5} GETHISTORY` | History by point code |
| `'Asset' ASSET` | Look up an asset |
| `{%%} GETEVENTS` | Events in range |

- `%%` is replaced with the display's range and grain (see `rangeformat`).
- A negative grain means a sample count. Time strings are UTC, `YYYY-MM-DD HH:mm:ss`.
- Endpoints:
  - `POST /aql/api/query` with `query`, `format=json`. Returns `{results:[{type:"pointlist", value:[...]}]}`.
  - `POST /aql/api/table` with `query`, `format=json`, `metadata=true`. Returns a table.

---

## 8. Official examples (raw source)

You can get any doc page as raw wiki text by inserting `_export/raw/` into its URL:
`https://ardi.com.au/docs/_export/raw/<namespace>:<page>`
To list the pages in a namespace, use `https://ardi.com.au/docs/doku.php?id=<ns>:welcome&do=index&idx=<ns>` and grep for `title="<ns>:`.

- **Tutorial** (`display_list:`): `writing_the_script`, `writing_the_script2`, `writing_the_script_3`, `writing_the_script4`, `writing_the_script5`, `adding_an_infographic`, `infographics_directory`, `folders`, `time-travel`, `third_party_libraries`, `3d`, `templates`, `creating_templates`, `embedding`, `marking_up_svgs`, `markup_examples`, `modifiers`.
- **Code examples** (`code:`; the Infographic index is `code:infographics`):
  - `barchart_of_all_properties`: live bar chart using `MapColour`, a min/max highlight and join transitions. The best live template.
  - `barchart_of_all_properties_across_a_relationship`
  - `simple_line_chart_of_a_property`, `interactive_line_chart`: history with `drawAxisSet`, a bisector tooltip and `getMouseAxisPos`.
  - `status-line_of_all_properties`: a discrete timeline using `MapValue` and `MapColour`.
  - `discrete_pie_or_donut_chart`, `pie_or_donut_charts`, `heatmap_basic`, `basic_area_chart`, `stacked_area_chart`, `normalised_stacked_area_chart`, `hbar_chart`, `basic_vbar_chart`, `colour_vbar_chart`, `vbar_minmax`.
  - The various `line_chart_-_*` pages are mostly for **Reports** (a separate system), not Infographics.

---

## 8a. Public demo server (useful for fixtures)

Optrix runs a public, read-only demo: `https://demo.optrix.com.au/s/pl` (a paint line) and `.../s/ex` (a wind farm).

- View a display: `https://demo.optrix.com.au/s/pl/displaylist/info?report=<name>&embed=true`. The page source shows the full bootstrap and the inlined `ActiveReport`.
- Libraries: `https://demo.optrix.com.au/addons/displaylist/reporting.js`, `/plugins/optrix/hmitt.js`, `/addons/displaylist/d3.min.js`, `/plugins/jquery/jquery.min.js`, CSS at `/s/pl/displaylist/online.css`.
- Live AQL works from curl, which is handy for building test fixtures:
  ```
  curl -s -X POST "https://demo.optrix.com.au/s/pl/aql/api/query" \
       --data-urlencode "query='Temperature - Oven' PROPERTY ALLPOINTS" --data-urlencode "format=json"
  ```
  The paint-line site has 7 `Temperature - Oven` points (Drying Oven, Prime/Finish Oven Zones 1–3, 0–400 Deg C).
