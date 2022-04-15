# FreemarkerGen tool

This tool is to test Freemarker templates that can be used in message tranformations in the opentunnel.

Minimal java version 1.8

## Usage

` java -jar ~/tools/FreemarkerGen_jre18.jar -a vars.txt -t template.ftl -x input.xml -o output.xml -g groovy/lib/`

### Options

```bash
version 0.3

usage: java -jar freemarker-gen.jar
 -a,--vars <arg>             vars input file path
 -f,--function-dir <arg>     Function dir
 -g,--groovy-lib-dir <arg>   groovy lib path
 -j,--json-input <arg>       json input filepath
 -o,--output <arg>           output file
 -r,--resolve <arg>          outputfile with resolved variables
 -s,--xsd-schema <arg>       xsd schema filepath
 -t,--template <arg>         freemarker template file
 -v,--validate               validate resulting file add schema if
                             applicable
 -x,--xml-input <arg>        xml input filepath
```

You can use either the xml-input or the json-input, never both.
The xml and json datastructures are made available through

- `payload`         this is a json string that can be parsed to a jsonmap.
- `payloadElement`  this is the objectrepresentation for the xml data structure.

## Debugging
When an error occurs in the template all known variables are dumped in the output file.
This way the user has an insight what the variables contained at the time of the error.

Also ther is a possibility to make the tool dump all variables at a specific point by adding
`${DEBUG}` at some point in your template. Then all variables are dumped in the outputfile at the 
point of the DEBUG statement.

You may add a variable to the DEBUG statement so only that variable is dumped to your output file.
`${DEBUG("varname")}`
when the first argument starts with a : the folowing character is used to format the debug string;
- `:l` debug format is single line only

`${DEBUG("varname")}` prints like

```
_________________ DEBUG info __________________
{
  "DebugVariables" : {
    "varname" : "NUL0"
  }
}
_______________ END DEBUG info ________________
```

`${DEBUG(":l","varname")}` prints like
```
DEBUG -> {"DebugVariables":{"i":"NUL0"}}
```
For simple variable structures the :l is more preferable.

## tunnelsvariables

You can add your tunnelvars to the vars.txt properties file

``name=value`` or ``name=const://value``

Use the content of a file as the value:

``name=file://filepath``

At this moment concatenation of tunnelvars is also allowed.

``name=const://valueconst://test`` is invalid

## tunnelfunctions

You may use tunnelfunctions by adding these to the vars.txt properties file.

``function://name=value``

Or use a groovy script, the value is then the file path to the script.

``function://name=groovy://filepath``

The groovy script has all the tunnelvars available as variables and also accepts parameters from you freemarker template. These parameters can be just a string or another tunnelvar.

``${tunnelFunction("helloworld",myname)}``

When you might need java jar libraries you may place them in a directory and use the -g option to define the path to that directory. The FreemarkerGen scans and load all jar files in that directory.

Look below for an example groovy script.

## XPath and jsonpath

When working with xml input files you may also use the xpath:// directive. ``varname=xpath://ns:element`` It should work just like in opentunnel.
The jsonpath works just like the xpath, the json payload is first transformed to an xml structure and then the xpath is resolved.


When the groovy tunnelfunction is used from the freemarker template the groovy script is then executed. The groovy engine has no access to any OpenTunnel specific classes 
so in the case you do use specific OT classes you can not test your groovy scripts using this tool.

## attachments

You may use attachements from the vars.txt properties file. You can just add a specific value to them or use the content of a specific file.

``attachment://name=file://filepath``

``attachment://name=value``

## multipart formdata

Multipart formdata that is not a file attachment is parsed as a ``url_`` tunnelvar. You may use a file or a vaue to parse.

``url_name=file://filepath``

``url_name=value``
 
 
## vars.txt properties file

The file contains a set of key/value pairs that normaly are configured in the opentunnel as tunnelvars. Or that are parsed to the template in the runtime.
 

```properties
#namespaces
xmlns://test=http://www.omgevingswet.nl/koppelvlak/stam-v2/IMAM

exitPoint=loopback.generic

businessTimestamp=20200908123049714
messageId=zWbs5X7ITY68gYPYH4iMmg@172.20.0.3
partyId=00000001805329535000
processId=8QHheAWCTBKxKGGYcZnCpA
service=NVT

timestamp=20200908123049714
timestampDay=8
timestampDay2=08
timestampHour=12
timestampMinute=30
timestampMonth=9
timestampMonth2=09
timestampSecond=49
timestampYear=2020

jsonfile=file://test.json

myname=Michiel
tunnelfunction://helloworld=groovy://test.groovy

tunnelfunction://boring=boring things happening
tunnelfunction://jsoninput=file://test.json

attachment://hello_json={"dom":"doen"}
attachment://hello_xml=<hello>xml</hello>
url_newfile=file://input.xml

xptest=xpath:////vx:type

```

### The Freemarker template.ftl

```freemarker
${tunnelFunction("helloworld",myname)}
${tunnelFunction("boring")}
${tunnelFunction("jsoninput")}
------------------------------------------------------------
get ALL available keys in the datamodel

<#assign formAttachments = {}>
<#assign multipart = {}>
<#list .data_model?keys as key>
  ${key}
  <#if key?matches("form.attachment.\\d+")>
    <#assign formAttachments += {key:.data_model[key]} >
  </#if>
  <#if key?starts_with("url_")>
  	<#assign multipart += {key:.data_model[key]}>
  </#if>
</#list>
------------------------------------------------------------
get ALL multipart data

<#list multipart?keys as key>
	${key}
	${multipart[key]}
=====
</#list>
------------------------------------------------------------
get ALL attachements by its names

<#list formAttachments?keys as key>
    <#assign att = attachments(messageheader,formAttachments[key])>
    <#list att?keys as ha>
=====
        <#list att[ha]?keys as hak>
            <#if att[ha][hak]?is_string || att[ha][hak]?is_boolean || att[ha][hak]?is_number || att[ha][hak]?is_date>
${hak} : ${att[ha][hak]}
            </#if>
        </#list>
=====        
    </#list>
</#list>
------------------------------------------------------------
get ALL attachments by their keys in attachments

<#assign allAttachments = attachments(messageheader)>
<#list allAttachments?keys as key>
keyname:     ${key}
partname:    ${allAttachments[key].partName}
contentType: ${allAttachments[key].contentType}
data:        ${allAttachments[key].data}
</#list>
```

### Example Groovy script

```groovy
class Greeter {
    String sayHello(name) {
        def greet = "Hello, Freemarkergen " + name
        greet
    }
}

def gr = new Greeter()

gr.sayHello(args[1])
```

### Example tasks for visual studio code

To use the freemarkergen tool from vscode and test your templates you need to add a folder named .vscode
In the .vscode folder create a file named tasks.json and add the following content.
Now when you have all files in place press ctrl-shift-B and freemarkergen will run with the options supplied in the tasks.json
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "current file",
      "type": "shell",
      "command": "java -jar ~/tools/Freemarkergen.jar -t ${fileBasename} -x verstrekVordering_voorbeeld.xml -a vars.properties -o ${fileBasenameNoExtension}.xml",
      "problemMatcher": [
        {
          "owner": "freemarkergen",
          "fileLocation": [
            "relative",
            "${workspaceFolder}"
          ],
          "pattern": {
            "regexp": ".*?template\\s\"(.*)\".*line\\s(\\d+).*column\\s(\\d+).*$",
            "file": 1,
            "line": 2,
            "column": 3
          },
          "severity": "error"
        }
      ],
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "presentation": {
        "clear": true
      }
    }
  ]
}

```