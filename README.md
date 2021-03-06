ELM327-emulator
===============

A Python emulator of the ELM327 OBD-II adapter connected to a vehicle.

*ELM327-emulator* provides a virtual serial communication port to client applications (via [pseudo-terminal](https://en.wikipedia.org/wiki/Pseudoterminal) function) and simulates an [ELM327](https://en.wikipedia.org/wiki/ELM327) adapter connected to a vehicle through the [OBD-II](https://en.wikipedia.org/wiki/On-board_diagnostics) protocol. It includes a command-line interface for extensive monitoring and controlling.

*ELM327-emulator* is agnostic of the client application accessing the serial port and has been tested with [python-OBD](https://github.com/brendanwhitfield/python-OBD).

An internal dictionary (named *ObdMessage*) allows configuring the emulation, which is set to reproduce the message flow generated by a Toyota Auris Hybrid car (through `scenario car` option), including custom PIDs and can be easily configured to statically and dynamically update its dictionary to simulate OBDII answers produced by other vehicles.


# Installation

```shell
# Checking Python version (should be 3.6 or higher)
python3.7 -V

# Installing prerequisites
python3.7 -m pip install pyyaml
python3.7 -m pip install obd # this is needed for obd_dictionary.py

# Downloading ELM327-emulator
git clone https://github.com/ircama/ELM327-emulator.git
cd ELM327-emulator
```

# Usage

The emulator allows batch and interactive mode. The latter is the default and can be executed as follows:

```shell
python3.7 -m elm
```

After starting the program, the emulator is ready to use. To enable the preconfigured set of PIDs of a Toyota Auris Hybrid car, enter `scenario car`.

The external application interfacing the emulator just needs to connect to the virtual device shown by the emulator and interact with the vehicle as if it was accessing a real ELM327 adapter.

All subsequent information are not needed for a basic usage of the tool and allow to master *ELM327-emulator*, exploiting it to test specific features including the simulation of communication exceptions, which are not always easy to be reproduced with a real link.


# Compatibility

ELM327-emulator has been tested with Python 3.6 and 3.7. Python 2 is not supported.

This code needs pseudo-terminal handling (pty support, `import pty`) which is platform dependent and runs on UNIX OSs. With Windows, [cygwin](http://www.cygwin.com/) is supported.


# Description

The serial port to be used by the application interfacing the emulator is displayed when starting the program. E.g.:

    ELM327-emulator is running on /dev/pts/0

## Embedded dictionary of AT Commands and PIDs

A [dictionary](https://docs.python.org/3.7/tutorial/datastructures.html#dictionaries) named *ObdMessage* is used to define commands and PIDs. The dictionary includes more sections (named scenarios):

- `'AT'`: set of default AT commands
- `'default'`: set of default PIDs
- `'car'`: PIDs of a Toyota Auris Hybrid vehicle
- any additional custom section can be used to define specific scenarios

Default settings include both the 'AT' and the 'default' scenarios.

The dictionary used to parse each ELM command is dynamically built as a union of three defined scenarios in the following order: 'default', 'AT', custom scenario (when applied). Each subsequent scenario redefines commands of the previous scenarios. In principle, 'AT' scenario is added to 'default' and, if a custom scenario is used, this is also added on top, and all equal keys are replaced. Then the Priority key defines the precedence to match elements.

If a custom scenario is selected through the *scenario* command, any key defined in the custom scenario replaces the default settings ('AT' and 'default' scenarios).

The key used in the dictionary consists of a unique identifier for each PID. Allowed values for each key (PID):

- `'Request'`: received data; a [regular expression](https://docs.python.org/3/library/re.html) can be used
- `'Descr'`: string describing the PID
- `'Exec'`: command to be executed
- `'Log'`: *logging.debug* argument
- `'ResponseFooter'`: run a function and returns a footer to the response (a [lambda function](https://docs.python.org/3/reference/expressions.html#lambda) can be used)
- `'ResponseHeader'`: run a function and returns a header to the response (a [lambda function](https://docs.python.org/3/reference/expressions.html#lambda) can be used)
- `'Response'`: returned data; can be a string or a list/tuple of strings; if more strings are included, the emulator randomly select one of them each time
- `'Action'`: can be set to 'skip' in order to skip the processing of the PID
- `'Header'`: if set, process the command only if the corresponding header matches
- `'Priority'=number`: when set, the key has higher priority than the default (highest number = 1, lowest = 10 = default)

The emulator provides a monitoring front-end, supporting commands and controlling the backend thread which executes the actual process.

## Built-in keywords

At the `CMD> ` prompt, the emulator accepts the following commands:

- `help` = List available commands (or detailed help with "help cmd").
- `quit` (or end-of-file/Control-D, or break/Control-C) = quit the program
- `counters` = print the number of each executed PIDs (upper case names), the values associated to some 'AT' PIDs (*cmd_...*), the unknown requests, the emulator response delay, the total number of executed commands (*commands*) and the current scenario (*scenario*). The related dictionary is `emulator.counters`.
- `pause` = pause the execution. (Related attribute is `emulator.threadState = THREAD.PAUSED`.)
- `prompt` = toggle prompt off/on if no argument is used, or change the prompt if using an argument
- `resume` = resume the execution after pausing; also prints the used device. (Related attribute is `emulator.threadState = THREAD.ACTIVE`)
- `delay <n>` = delay each emulator response of `<n>` seconds (floating point number; default is 0.5 seconds)
- `wait <n>` = delay the execution of the next command of `<n>` seconds (floating point number; default is 10 seconds)
- `engineoff` = switch to *engineoff* scenario
- `scenario <scenario>` = switch to `<scenario>` scenario; if the scenario is missing or invalid, defaults to `'car'`. The autocompletion (by pressing or double-pressing TAB) allows prompting all compatible scenarios defined in `emulator.ObdMessage`. (Related attribute is `emulator.scenario`.)
- `default` = reset to *default* scenario
- `reset` = reset the emulator (counters and variables)
- `color` = toggle usage of colors off/on
- `history [<n>]` = print the last 20 items of the command history; if an argument is given, print the last n items in the history; with argument *clear*, clears the history. The command history is permanently saved to file *.ELM327_emulator_history* within the home directory.
- `merge <module>` = import a scenario from an external module and merges it with the emulator configuration. `<module>` shall be a Python file including the `ObdMessage` dictionary (e.g., generated by *obd_dictionary.py*) without *.py* extension (notice that the physical file shall be in the current directory and shall end with *.py*). The autocompletion allows prompting all compatible files in the current directory (type *merge*, then press space, then press TAB). After a successful merge, the new scenario can be activated through the *scenario* command.

In addition to the previously listed keywords, any Python command is allowed to query/configure the backend thread.

At the command prompt, cursors and [keyboard shortcuts](https://github.com/chzyer/readline/blob/master/doc/shortcut.md) are allowed. Autocompletion (via TAB key) is active for all previously described commands and also allows Python keywords and namespaces (built-ins, self and global). If the autocompletion matches a single item, this is immediately expanded; Conversely, if more possibilities are matched, none of them is returned, but pressing TAB again a list of available options is displayed.

## Advanced usage

*echo* and *linefeed* settings are both disabled by default. They can be configured via related AT commands (*ATL1* and *ATE1*). To enable them via command line:
```
emulator.counters['cmd_linefeeds'] = True; emulator.counters['cmd_echo'] = True
```

Space characters are inserted by default in the ECU response as per specification. To remove them, use the AT command *ATS0* or `emulator.counters['cmd_spaces']=0`, Notice that an *ATZ* command resets all counters.

The emulator includes a timeout management for each entered character, which by default is not active (e.g., set to 1440 seconds). This setting can be configured through `emulator.counters['req_timeout']`. Decimals are allowed. Some adapters provide a feature that discards characters if each of them is not entered within a short time limit (apart from the first one after a CR/Carriage Return). The appropriate emulation for this timeout is to set `emulator.counters['req_timeout']=0.015` (e.g., 15 milliseconds). Typing commands by hand via terminal emulator with such adapters is not possible as the allowed timing is too short. The same happens when setting *req_timeout* to 0.015.

The command prompt also allows configuring the `emulator.answer` dictionary, which has the goal to dynamically redefine answers for specific PIDs (`'Pid': '...'`). Its syntax is:

```python
emulator.answer = { 'pid' : 'answer', 'pid' : 'answer', ... }
```

Example:

```python
emulator.answer = { 'SPEED': 'NO DATA\r', 'RPM': 'NO DATA\r' }
# Or, alternatively:
emulator.answer['SPEED']='NO DATA\r'
emulator.answer['RPM']='NO DATA\r'
```

The above example forces SPEED and RPM PIDs to always return "NO DATA".

To reset the *emulator.answer* string to its default value:

```python
emulator.answer = {}
# Or, alternatively:
del emulator.answer['SPEED']
del emulator.answer['RPM']
```

To simulate that the adapter is not connected to the vehicle:

```python
emulator.answer['AT_R_VOLT'] = '0.0V'
```

This dictionary can be used to modify answers within a workflow. The front-end allows implementing basic Python workflows and, when used in batch mode, can also be controlled by a piped external supervisor. The following examples show some simple workflows in interactive mode.

Example of automation which suspends the emulator for 10 seconds:

```python
emulator.threadState = THREAD.PAUSED; time.sleep(10); emulator.threadState = THREAD.ACTIVE
```

Example of an automation that simulates the off/on ignition states:

```python
CMD> for i in range(10): emulator.scenario="car" if i % 2 else "engineoff"; print(emulator.scenario); time.sleep(10)
engineoff
car
engineoff
car
engineoff
car
engineoff
car
engineoff
car
```

## Configuring response strings

Response strings allow embedding Python statements and expressions. Specifically, `Response`, `ResponseHeader`, `ResponseFooter` and `emulator.answer` support single and multiple in-line Python commands (expressions or statements) when embraced between `\0` tags: this feature for instance can be used to embed real-time delays between strings or to differentiate answers. The return value of a statement is ignored. The evaluation of an expression is substituted. Spaces inside `\0` tags are allowed and can be used to improve readability. Example: `'Response' = 'SEARCHING...\0 time.sleep(1) \0\rUNABLE TO CONNECT\r'`. This returns `SEARCHING...`, then waits one second, then returns `\rUNABLE TO CONNECT\r`. Notice that, as `time.sleep` is a statement, the related return value is ignored.

Further processing can be achieved through a *lambda function* applied to `ResponseHeader`, `ResponseFooter`. It has to manage the following parameters: *self*, *cmd*, *pid*, *val* (e.g., `lambda self, cmd, pid, val:`).

- cmd: the request, received by the client application
- pid: the PID identifier (which can be used as key to index `self.counters` and `ObdMessage`)
- val: `ObdMessage` value related to `pid` (e.g., `val['Response']`).

Example of PID definition within the `ObdMessage` dictionary:

```python
            'ELM_PIDS_A': {
                'Request': '^0100$',
                'Descr': 'PIDS_A',
                'ResponseHeader': \
                lambda self, cmd, pid, val: \
                    'SEARCHING...\0 time.sleep(1) \0\rUNABLE TO CONNECT\r' \
                    if self.counters[pid] == 1 else 'NO DATA\r',
                'Response': '',
                'Priority': 5
            },
```

In the above example, the first time *ResponseHeader* is executed, the produced response is `SEARCHING...`, followed by a one-second delay and then `\rUNABLE TO CONNECT\r`. For all subsequent messages, the response will be different and produces `'NO DATA\r'`.

The ability to add dynamic differentiators and delays within responses enables testing specific use cases and exceptions that are difficult to be achieved through a real connection with a car. These not only apply to the `ObdMessage` dictionary (by editing *obd_message.py*), but also to `emulator.answer`, that can be configured through the command line. Consider for instance the following dynamic configuration via command line:

```python
emulator.answer['SPEED'] = '\0 self.ECU_R_ADDR_E + " 03 41 0D 0A " if randint(0, 100) > 20 else "NO DATA" \0\r'
```

In the above example, which illustrates an in-line expression substitution, the configuration of the ‘SPEED’ PID is replaced with a dynamic answer and the ‘SPEED’ PID will return `7E8 03 41 0D 0A` + newline for most of the times. With 20% probability, `NO DATA` + newline is returned. Notice that the last `\r` is common to both options. (Notice also that ECU headers shall by referenced within the *self* namespace.)

To list the configuration, type `emulator.answer`, or simply `counters`. To remove the dynamic answer and return to the default configuration of the ‘SPEED’ PID, type `del emulator.answer['SPEED']`.

## Logging and monitoring

Logging is controlled through `elm.yaml` file (in the current directory by default). Its path can be set through the *ELM_LOG_CFG* environment variable.

The logging level can be dynamically changed through `logging.getLogger().handlers[n].setLevel()`. To check that *console* is the first handler (e.g., `handlers[0]`), run `for n, l in enumerate(logging.getLogger().handlers): print(n, l.name)`. For instance, if *console* refers to the first handler (default settings of the provided `elm.yaml` file), the following commands will change the logging level:

```python
logging.getLogger().handlers[0].setLevel(logging.DEBUG)
logging.getLogger().handlers[0].setLevel(logging.INFO)
logging.getLogger().handlers[0].setLevel(logging.WARNING)
logging.getLogger().handlers[0].setLevel(logging.ERROR)
logging.getLogger().handlers[0].setLevel(logging.CRITICAL)
```

It is possible to add marks in the log file via commands like `logging.info("my mark")`

To totally disable logging for all handlers: `logging.disable(logging.CRITICAL)`. To restore logging: `logging.disable(0)`.

Command to count the number of different PIDs (OBD Commands) used by the client (excluding AT Commands):
```python
import re
from functools import reduce
reduce(lambda x, key: x + (1 if re.match('^[A-Z]', key) and not key.startswith('AT_') and emulator.counters[key] > 0 else 0), emulator.counters, 0)
```

The following command returns the total number of OBD Commands (PID queries issued by the client excluding AT Commands):
```python
import re
from functools import reduce
reduce(lambda x, key: x + (emulator.counters[key] if re.match('^[A-Z]', key) and not key.startswith('AT_') else 0), emulator.counters, 0)
```

To only count AT Commands:
```python
from functools import reduce
reduce(lambda x, key: x + (emulator.counters[key] if key.startswith('AT_') else 0), emulator.counters, 0)
```

Print the average number of processed commands per second within a 5 seconds period:
```python
a=emulator.counters['commands'];time.sleep(5);print((emulator.counters['commands']-a)/5)
```

To save a CSV file including the *emulator.counters* dictionary:
```python
with open('mycounters.txt', 'w') as f: f.write('\r\n'.join([x + ', ' + repr(emulator.counters[x]) for x in emulator.counters]))
```

## ObdMessage Dictionary Generator for "ELM327-emulator" (obd_dictionary)

*obd_dictionary* is a dictionary generator for "ELM327-emulator".

It queries the vehicle via *python-OBD* for all available commands and is also able to process custom PIDs described in [Torque CSV files](https://torque-bhp.com/wiki/PIDs).

Its output is a Python *ObdMessage* dictionary that can either replace the *obd_message.py* module of *ELM327-emulator*, or extend the existing dictionary via *merge* command, so that the emulator will be able to provide the same commands returned by the vehicle.

Notice that querying the vehicle might be invasive and some commands can change the car configuration (enabling or disabling belts alarm, enabling or disabling reverse beeps, clearing diagnostic codes, controlling fans, etc.). In order to prevent dangerous PIDs to be used for building the dictionary, a PID blacklist (*blacklisted_pids*) can be edited in *elm.py*. To check all PIDs without performing actual OBDII queries (dry-run mode), use the `-p 0` option (the standard error output with default logging level shows the list of produced PIDs).

```
usage: obd_dictionary.py [-h] -i DEVICE [-c CSV_FILE] [-o FILE] [-v] [-V]
                         [-p PROBES] [-d DELAY] [-D DELAY_COMMANDS]
                         [-n CAR_NAME] [-b] [-t [FILE]] [-m]

optional arguments:
  -h, --help            show this help message and exit
  -i DEVICE             serial port connected to the ELM327 adapter (required
                        argument)
  -c CSV_FILE, --csv CSV_FILE
                        input csv file including custom PIDs (Torque CSV
                        Format: https://torque-bhp.com/wiki/PIDs) '-' reads
                        data from the standard input
  -o FILE, --out FILE
                        output dictionary file generated after processing
                        input data (replaced if existing). Default is to print
                        data to the standard output
  -v, --verbosity       print process information
  -V, --verbosity_debug
                        print debug information
  -p PROBES, --probes PROBES
                        number of probes (each probe includes querying all
                        PIDs to the OBDII adapter)
  -d DELAY, --delay DELAY
                        delay (in seconds) between probes
  -D DELAY_COMMANDS, --delay_commands DELAY_COMMANDS
                        delay (in seconds) between each PID query within all
                        probes
  -n CAR_NAME, --name CAR_NAME
                        name of the car (dictionary label; default is "car")
  -b, --blacklist       include blacklisted PIDs within probes
  -t [FILE], --at [FILE]
                        include AT Commands within probes. If a dictionary
                        file is given, also extract AT Commnands from the
                        input file and add them to the output
  -m, --missing         add in-line comment to dictionary for PIDs with
                        missing response
```

Sample usage: `obd_dictionary.py -i /dev/ttyUSB0 -c car.csv -o ObdMessage.py -v -p 10 -d 1 -n mycar`

In general, *ELM327-emulator* should already manage all needed AT Commands within its default dictionary, so in most cases it is worthwhile removing them from the new scenario via `-t` option.

The file produced by *obd_dictionary.py* provides the same information model of *obd_message.py*. It can be used to replace the default module or can be dynamically imported in *ELM327-emulator* through the `merge` command, which loads an *ObdMessage* dictionary and merges it to *emulator.ObdMessage*. Example of *merge* process:

```python
# Create AurisOutput.py
python3 obd_dictionary.py -i /dev/ttyUSB0 -c auris.csv -o AurisOutput.py -n Auris
python3 -m elm # run ELM327-emulator
merge AurisOutput
scenario Auris
```

To help configuring the emulator, autocompletion is allowed (by pressing TAB) when prompting the `merge` command, including the `merge` argument. Also variables and keywords like `scenario` accept autocompletion, including the `scenario` argument.

A merged scenario can be removed via `del emulator.ObdMessage['<name of the scenario to be removed>']`.

To produce a complete dictionary file that can replace *obd_dictionary.py*:

```python
python3 obd_dictionary.py -i /dev/ttyUSB0 -c auris.csv -o obd_dictionary.py -n default -t elm/obd_message.py
```

## ELM327-emulator batch mode

*ELM327-emulator* can be run in batch mode to allow automating tests and background execution. The `-b FILE` option allows this mode and writes the output to FILE. The first line in that file will be the virtual serial device, which can be read to a shell variable through `read variable_name < output_file`. Commands can be piped in (e.g., within a bash script) to configure the emulator (e.g., via `echo -e`). The appropriate way to kill a background instance of the emulator is with the SIGINT signal (`kill -2`). To ensure that the external application is started only after correct setup of the emulator, the input commands can be terminated with a string (e.g., "RUNNING") that can then be recognised before starting the application.

The description of the *ELM327-emulator* command-line option is the following:

```
Usage: python3 -m elm [-h] [-b FILE]

optional arguments:
  -h, --help            show this help message and exit
  -b FILE, --batch FILE
                        Run ELM327-emulator in batch mode. Argument is the
                        output file. The first line in that file will be the
                        virtual serial device

 - ELM327 OBDII adapter emulator
 ```

The following script shows an example of batch mode usage. *obd_dictionary.py* is run after starting *ELM327-emulator* in background and is used here as example of external application interfacing the emulator. The output of the emulator is saved to $FILE and the background process id is saved to $EMUL_PID.

 ```bash
FILE=/tmp/elm$$
echo -e 'scenario car\ncounters\n"RUNNING"' | python3 -m elm -b $FILE &
EMUL_PID=$!

until grep "^RUNNING$" $FILE; do sleep 0.5; done
read TTYNAME < $FILE

../obd_dictionary.py -i /dev/pts/0 -o $TTYNAME -t -v -o /dev/null

kill -SIGINT $EMUL_PID
cat $FILE
rm $FILE
```
