# monitoring-plugin-XML-RFC
A "Request for Commects" on an XML Representation of monitoring plugin output
RFC: An XML representation for the output of monitoring plugins

Introduction
------------

For background see [Monitoring Plugins Development
Guidelines](https://www.monitoring-plugins.org/doc/guidelines.html).

After having developed some sophisticated in-house monitoring plugins for
Centreon/Nagios, management decided to switch to an incompatible product.
So I was wondering whether to re-write (and re-debug) the plugins to produce
the desired output format.

While studyin the new output format I realized that the format is terribly
designed, so I refused to rewrite my nice plugins for the poorly designed
output format.

Instead I had the idea to convert the Nagios output format to the needed
format, keeping the original plugins untouched.

So the first step was to parse the output into some structure, and for
debugging purposes I wanted to show that structure as well.
While my converter was written in Perl, and the data structure was some
combination of nested arrays and hashes, I wanted to define some XML DTD to
represent the result of parsing.  Alternatively I had JSON in mind.

Using XML to represent the monitoring status would also allow the use of tools
like XSLT to transform the structure to anything else.
At that time of development I did not care about performance, however.

Elements of a Monitoring Status
-------------------------------

To help understanding the following, here's a description of the elements
identified in the status output of a monitoring plugin (alphabetically):

**crit_labels**: Array with those labels from the performance data that triggered
a critical condition.  Set after having processed any thresholds and
performance data output.

**description**: A textual description (multiple words) for the object.

**exit_code**: The plugin's exit code.

**id**: The ID is a simple word used to uniquely identify the object (among a
limited set).

**info_string**: The info part of the output (following the status part).

**perf_data**: The temporary internal representation of perf_string created by
parse_perfdata.  Not intended to be used directly.

**perf_labels**: Internal list containing the ordered labels of perf_labels.  Not
intended to be used directly.  Needed as ordering of Perl hashes is
undetermined, and some tools expect the performance data labels to appear in a
fixed order.

**perf_string**: The performance data part of the output (following |).

**status_string**: The status part of the output. Typically it contains words like
OK or WARNING, related to the plugin's exit status.

**warn_labels**: Array with those labels from the performance data that triggered
a warning condition.  Set after having processed any thresholds and
performance data output.


An Example using JSON Output
----------------------------

Given this hypothetical plugin output including performance data:

	OK: (demo): Some interesting story|L1=0.49 L2=4.7s;;;0;7 L3=0.6;@0.2:0.3

The JSON output (pretty-printed) will look like this:

~~~lang-json
{
    "crit_labels":[],
    "description":"This is a test output",
    "perf_data":{
        "L1":{
            "value":"0.49"
        },
        "L2":{
            "unit":"s",
            "range":{
                "min":"0",
                "max":"7"
            },
            "value":"4.7"
        },
        "L3":{
            "value":"0.6",
            "thresholds":{
                "warn":{
                    "start":"0.2",
                    "inverted":1,
                    "end":"0.3"
                }
            }
        }
    },
    "info_string":"(demo): Some interesting story",
    "warn_labels":[],
    "perf_string":"L1=0.49 L2=4.7s;;;0;7 L3=0.6;@0.2:0.3",
    "status_string":"OK",
    "id":"test-id",
    "exit_code":0
}
~~~

The Monitoring Output XML DTD
-----------------------------

So after some experiments I came up with this XML DTD:

~~~lang-xml
<?xml version="1.0" encoding="utf-8"?>
<!-- XML DTD for MonitoringOutput -->
<!-- $Id$ -->
<!-- Note: The "and connector" (&) is not available for XML DTDs -->
<!ELEMENT MonitoringOutput (description? , exit_code? , status_string? ,
	  info_string? ,  perf_string? , perf_data?)>
<!ATTLIST MonitoringOutput
	  id CDATA ""
	  version CDATA #REQUIRED>

<!-- MonitoringOutput -->
<!ELEMENT description (#PCDATA)>
<!ATTLIST description>

<!ELEMENT exit_code (#PCDATA)>
<!ATTLIST exit_code>

<!ELEMENT info_string (#PCDATA)>
<!ATTLIST info_string>

<!ELEMENT perf_string (#PCDATA)>
<!ATTLIST perf_string>

<!ELEMENT perf_data (sample+, warn_labels?, crit_labels?)>
<!ATTLIST perf_data
	  count CDATA #REQUIRED>

<!ELEMENT status_string (#PCDATA)>
<!ATTLIST status_string>

<!ELEMENT label (#PCDATA)>
<!ATTLIST label>

<!-- perf_data -->
<!ELEMENT crit_labels (name+)>
<!ATTLIST crit_labels
	  count CDATA #REQUIRED>

<!ELEMENT sample (label, value?, unit?, thresholds?, range?)>
<!ATTLIST sample
	  label CDATA #REQUIRED>

<!ELEMENT range EMPTY>
<!ATTLIST range
	  min CDATA "0"
	  max CDATA "0">

<!ELEMENT thresholds (warn? , crit?)>
<!ATTLIST thresholds>

<!ELEMENT unit (#PCDATA)>
<!ATTLIST unit>

<!ELEMENT value (#PCDATA)>
<!ATTLIST value
	  unit CDATA ''>

<!ELEMENT warn_labels (name+)>
<!ATTLIST warn_labels
	  count CDATA #REQUIRED>

<!-- crit_labels, warn_labels -->
<!ELEMENT name (#PCDATA)>
<!ATTLIST name>

<!-- thresholds -->
<!ENTITY % threshold "end CDATA '' inverted CDATA '' start CDATA ''">
<!ELEMENT crit EMPTY>
<!ATTLIST crit
	  %threshold;>

<!ELEMENT warn EMPTY>
<!ATTLIST warn
	  %threshold;>
~~~

And using the example shown earlier, the XML (pretty-printed) output would
look like this:

~~~lang-xml
<?xml version="1.0" encoding="utf-8"?>
<MonitoringOutput id="id-14024" version="0.2">
  <description>demo.out</description>
  <exit_code>0</exit_code>
  <status_string>OK</status_string>
  <info_string>(demo): Some interesting story</info_string>
  <perf_string>L1=0.49 L2=4.7s;;;0;7 L3=0.6;@0.2:0.3</perf_string>
  <perf_data count="3">
    <sample label="L3">
      <label>L3</label>
      <value>0.6</value>
      <thresholds>
        <warn end="0.3" inverted="1" start="0.2"/>
      </thresholds>
    </sample>
    <sample label="L1">
      <label>L1</label>
      <value>0.49</value>
    </sample>
    <sample label="L2">
      <label>L2</label>
      <value unit="s">4.7</value>
      <unit>s</unit>
      <range max="7" min="0"/>
    </sample>
  </perf_data>
</MonitoringOutput>
~~~

To allow single-pass parsing, the XML output has some wanted redundency;
for example the "count" attribute for "perf_data" helps pre-allocating a array
to hold the "sample" children to follow.

A more complex example with a non-success status follows:

The original nagios output was:

    CRITICAL: [1/4 Databases], [CRIT DB3.PagesFreeMinPct: 0.00391 < 10], [CRIT DB3.PagesFreeMaxPct: 3.4805 < 10], [2 searches with 5 entries in 0.054s], Time.Uptime=611748, DB3.PagesFreeMinPct=0.00391%, DB3.PagesFreeMaxPct=3.4805%|DB3.PagesFreeMinPct=0.00391%;15:;10:;0;100 DB3.PagesFreeMaxPct=3.4805%;15:;10:;0;100

And here's the XML representation:

~~~lang-xml
<?xml version="1.0" encoding="utf-8"?>
<MonitoringOutput id="id-13947" version="0.2">
  <description>/tmp/test.out</description>
  <exit_code>0</exit_code>
  <status_string>CRITICAL</status_string>
  <info_string>[1/4 Databases], [CRIT DB3.PagesFreeMinPct: 0.00391 &lt; 10], [CRIT DB3.PagesFreeMaxPct: 3.4805 &lt; 10], [2 searches with 5 entries in 0.054s], Time.Uptime=611748, DB3.PagesFreeMinPct=0.00391%, DB3.PagesFreeMaxPct=3.4805%</info_string>
  <perf_string>DB3.PagesFreeMinPct=0.00391%;15:;10:;0;100 DB3.PagesFreeMaxPct=3.4805%;15:;10:;0;100</perf_string>
  <perf_data count="2">
    <sample label="DB3.PagesFreeMinPct">
      <label>DB3.PagesFreeMinPct</label>
      <value unit="%">0.00391</value>
      <unit>%</unit>
      <thresholds>
        <warn end="~" start="15"/>
        <crit end="~" start="10"/>
      </thresholds>
      <range max="100" min="0"/>
    </sample>
    <sample label="DB3.PagesFreeMaxPct">
      <label>DB3.PagesFreeMaxPct</label>
      <value unit="%">3.4805</value>
      <unit>%</unit>
      <thresholds>
        <warn end="~" start="15"/>
        <crit end="~" start="10"/>
      </thresholds>
      <range max="100" min="0"/>
    </sample>
    <warn_labels count="2">
      <name>DB3.PagesFreeMinPct</name>
      <name>DB3.PagesFreeMaxPct</name>
    </warn_labels>
    <crit_labels count="2">
      <name>DB3.PagesFreeMinPct</name>
      <name>DB3.PagesFreeMaxPct</name>
    </crit_labels>
  </perf_data>
</MonitoringOutput>
~~~

So there are non-empty "warn_labels" and "crit_labels" elements with "name"
children each.  Again for single-pass parsing there is a "count" attribute
with the number of children.

Request for Comments
--------------------

What do you think about the structure (DTD)?

Ideas
-----

If the parser can check thresholds, the `warn_labels` and `crit_labels` could include the reason for the status.
