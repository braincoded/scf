# SCF service

The SCF service allows to convert subtitles between EBU STL and
different EBU-TT profiles (and vice versa). The conversion is done using
IRT's Subtitle Conversion Framework (SCF) which consists of different
conversion modules. These modules allow to convert a subtitle file
step-by-step to the desired target format/profile.

A Docker file is provided that builds an image containing SCF and BaseX
which provides a RestXQ service. Also a simple web interface is
available that allows to execute a conversion.

The following subtitle formats/profiles are supported:
- STL
- STLXML (an SCF internal intermediate format)
- EBU-TT
- EBU-TT-D
- EBU-TT-D-Basic-DE


## Building the Docker image

Just execute the following command to build the Docker image:

    docker build -t scf_service .


## Run the Docker image

To run the image, the web interface port (8984) has to be mapped to a
local port (here: 9000).

This can be achieved with the following command:

    docker run -p 9000:8984 scf_service


## Run standalone

The SCF service can also be used without Docker. This requires [BaseX](http://basex.org) to
be installed (tested with version 9.3), including [Saxon](http://saxon.sf.net) (HE) 9 (tested
with version 9.9.1.6). Hereby Saxon's `saxon9he.jar` has to be copied to
the `lib/custom` subfolder of the BaseX installation. Furthermore
Python 3 must be available.

The following BaseX command has then to be executed in the repository
root folder to start the service (which will be available on port 8984):

	basexhttp


## Web interface

The web interface is available at:

    localhost:9000/webif


### Usage

The mentioned web interface provides simple forms for common conversions
and also provides a form for a custom source/target format combination.
After a source file has been selected, the process can be started. When
the process is finished, the result is provided as download and can be
saved to a file by the user. Thereby an appropriate filename is
automatically proposed by the service, based on the source file's name.


## REST interface

A conversion can also be executed using the REST interface. The process
is started using the `convert` POST request and returns the result.

Note that Cross-Origin Resource Sharing (CORS) is enabled i.e. requests
from any origin are processed. This behaviour can be disabled by
removing the paragraph related to CORS in `/webapp/WEB-INF/web.xml`.

### `convert` POST request

This request executes a subtitles conversion and returns the conversion
result. If an error occurs, detailled error information is provided
instead.

The following request parameters are available:
- `input`: The source file (as `multipart/form-data`) to be converted.
- `format_source`: The format of the source file.
- `format_target`: The desired target format.
- `offset_seconds`: If present, the offset in seconds (e.g. `36000`)
  that shall be subtracted from all timecodes during conversion.
- `offset_frames`: If present, the offset in frames (e.g. `10:00:00:00`)
  that shall be subtracted from all timecodes during conversion.
- `offset_start_of_programme`: If present, the Start-of-Programme offset
  shall be subtracted from all timecodes during conversion.
- `separate_tti`: If present, multiple text TTI blocks of the same
  subtitle will not be merged.
- `clear_uda`: If present, the STL User-Defined Area (UDA) is cleared.
- `discard_user_data`: If present, present STL User Data is discarded.
- `use_line_height_125`: If present, use the value `125%` (instead of
  the special value `normal`) for the line height in EBU-TT-D.
- `ignore_manual_offset_for_tcp`: If present, any manual offset (seconds
  or frames) will *not* be subtracted from the TCP value.
- `indent`: If present, the output will be indented in case of a target
  format based on XML.

For the two format fields, the following values are supported:
`stl`, `stlxml`, `ebu-tt`, `ebu-tt-d`, `ebu-tt-d-basic-de`.

Note that option fields may not be supported for all possible conversion
chains. If an option is not supported, the conversion will abort with a
descriptive error message then.

On success, the response is in the related file format (as specified by
the transmitted media type in the response header), depending on the
target subtitle format, i.e. either binary data or XML. The HTTP status
code is `200 OK`.

If an error occurs during the conversion, further details are returned
(with status code `400 Bad Request`) as part of the following XML
structure:

```xml
<error>
  <steps>
    <step>{step}</step>
    ...
  </steps>
  <code>{code}</code>
  <description>{description}</description>
  <value>{value}</value>
  <module>{module}</module>
  <line-number>{line-number}</line-number>
  <column-number>{column-number}</column-number>
</error>
```

The following field values are transmitted (but not always filled!):
- `steps`: All conversion formats until (and including) the one at which
  the error occurs.
- `step`: A conversion format (see above).
- `code`: Error code of the catched exception.
- `description`: Description of the catched exception.
- `value`: Error value of the catched exception.
- `module`: Module in which the catched exception occured.
- `line-number`: Line number within the affected module.
- `column-number`: Column number within the affected line.

On error, in addition a processing instruction provides the relation to
a corresponding XSLT that creates a simple optical presentation of the
error details.


## Operation

The web and REST interfaces are provided using [BaseX](http://basex.org/) together with
RestXQ. The [Saxon (HE) processor](http://saxon.sf.net) is used as well, as it is able to
process XSLT versions newer than 1.0. The actual conversion is done 
using [IRT's SCF](https://github.com/IRT-Open-Source/scf/). The particular SCF modules use XSLT, XQuery and
Python 3.


### Details

The following files are relevant for the actual application:
- `.basexhome`: empty BaseX helper file to indicate home directory.
- `modules`: the SCF modules
- `webapp/scf_service.xqm`: application source code as XQuery module.
- `webapp/static/error.xsl`: XSLT used for rendering by the error result page.
- `webapp/WEB-INF/jetty.xml`: Jetty web server config
- `webapp/WEB-INF/web.xml`: web application config

A finite-state machine (FSM) is used to convert the subtitles
step-by-step from the source format to the target format. The FSM's
transitions are determined by a transition function. This function is
provided with the current format and the target format, and returns the
next format to which the subtitles shall (and can) be converted, towards
the target format. After all the necessary transitions/conversions,
finally the subtitles are available in the desired target format.
Depending on success/failure and the target format of the conversion,
the different header fields are set accordingly.

Most of the SCF modules are implemented in XSLT or XQuery. Such modules
can be natively invoked by the SCF service. Thus no temporary files are
required to perform the actual conversion. The module `STL2STLXML`
however is implemented in Python and requires to execute a Python
interpreter in a separate process. Furthermore the input/output data has
to be stored in (temporary) files, in order to prevent problems
regarding character encoding.

In case of a conversion option, the option is part of the status that is
propagated through all conversion steps. In case a step supports an
option, it forwards the option value as a parameter to the actual 
conversion operation. To indicate that the option has been applied, the
option value is cleared afterwards to prevent to apply it at another
step that supports this option, too.

After all necessary conversion steps have been executed, it is ensured
that no longer any option values are present - either because an option
has been cleared after being applied or because it has not been present
in the first place. If any option values are still present, at least one
option could not have been applied. Thus the conversion is aborted with
a corresponding error message.