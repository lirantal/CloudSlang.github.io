#SLANG - score language

##What is SLANG?

SLANG is a YAML based language for describing a workflow.

YAML is a human friendly data serialization standard for all programming languages. The acronym stands for
"YAML Ain't Markup Language" suggesting that its purpose is rather data-oriented than document markup. The YAML
(and SLANG) syntax maintains data structure hierarchy by outline indentation (in other words parallel elements should have the same left indentation).

SLANG as a workflow language is used to define processes introducing the concept of flows. See [SLANG DSL](#docs/#slang_dsl).
Using SLANG you can easily define your workflow in a structured, easy-to-understand format.

##SLANG DSL
The SLANG DSL contains the following main entities:

+ Flow: A flow is the basic executable unit of SLANG. It represents a process that can perform a job relevant to the end user. For example, a flow that creates a virtual machine. A flow consists of tasks and the navigation between them. Flows have inputs, outputs and possible results.
+ Task: One step in a flow. It can point to an operation or to another flow (which in this context is called subflow).
+ Operation: A wrapper unit of an action. The operation handles the action inputs, defines relevant outputs, and returns a result.
+ Action: A method, written pragmatically. This can be a java method or a type of script.
+ Result: The returned status of a flow or operation, for example: SUCCESS or FAILURE.

###Flow

A flow is the basic executable unit of SLANG. It represents a process that can be described in a number of ways and that can perform a job relevant to the end user. Each flow consists of a workflow describing the exact tasks (one or more) that are performed during the execution a flow. A flow can be used by other flows as a single task.

Property    |Required    |Default            |Description
------------|-----------|-------------------|-----------
name        |V          |                   |The name of the flow
inputs    	|	        |                   |List of inputs (see [Inputs](#docs/#inputs))
workflow	|V		    |                   |Describes the flow tasks and navigation (see [Workflow](#docs/#workflow))
results		|           |SUCCESS/FAILURE    |Possible results of the flow (see [Results](#docs/#results))
outputs		|   	    |                   |List of outputs (see [Outputs](#docs/#outputs))

*sample:*

```yaml
namespace: slang.sample.flows
imports: 
- ops: slang.ops.operations
flow:
  name: SimpleFlow
  inputs:
  - input_1
  - input_2
  - input_3
  workflow:
    CheckWeather:
      do: 
        ops.check_Weather:
          city: input_1
      publish:
        weather
    PrintWeather:
      operation:
        ops.print:
         - text:  "'the weather in' + input_1 + ' is:' + weather"
```


###Workflow

Describes the workflow of the flow, and contains the different tasks and the navigations between them.

The first task in the workflow is the begin task of the flow.
on_failure is a reserved key.
For the default navigation the result of FAILURE goes to on_failure, the result of SUCCESS goes to the next step.
You can also define custom navigations with targets like: another task or flow results (e.g. SUCCESS / FAILURE).
Every task can use: predefined operation or subflow, (see [Task](#docs/#task)).

Property	|Required	|Default	|Description
------------|-----------|-----------|-----------
on_failure	|		    |           |the default task FAILURE result should navigate to

*sample:*

```yaml
workflow:
  first_task:
    do: 
      some_operation:
      - operation_input_1: flow_input_3
      - operation_input_2
      - operation_input_3
    publish:
      flow_var: some_op_output
    navigate: 
      #how to handle an operation that has a non-default result
      #we'll use the default navigation as well: FAIL -> on_failure, SUCCESS -> next task (in our case second_task)
      NON_DEFAULT_RESULT: custom_task
      
```


###Inputs

Inputs are used to pass and manipulate parameters in flows, operations and tasks.
The name of the input is the key.

| Property | Required | Default | Description |
|----------|----------|---------|-------------|
|required  |          |true     |If the input must have a value|
|default|||The default value of the input (constant value or expression) in case no other value passed|
|encrypted||false|If the value is true, the input will be encrypted|
|override||false|In case it is marked as true, the default value is always the value - even if the parent task/flow declared the input with given value |

*sample:*

```yaml
	inputs:
	- input_with_value_like_name
	- input_not_required:
	   required: false
	- input_use_expression_inline: "'1' + '6'"
	- input_with_default_value:
	   default: "'I'm the default value'"
	- input_with_default_expression_value:
	   default: "'1' + '5'"
	- input_mix:
	   default: "'some value'"
	   required: false
```


###Outputs

Outputs define the possible parameters flow / operation exposed to further use (see [Task](#docs/#task) publish property).

*sample:*

```yaml
	outputs:
	- output_from_return_value: processId
	- output_from_input_value: fromInputs ['input1']
```


###Results

Describes the possible results of the flow. By default, the flow has two results: SUCCESS and FAILURE. You can override them, with an unlimited number of results. The result on runtime will be used to navigate when using this flow as sub-flow in a different flow.
*sample:*

```yaml
	results:
	 - SUCCESS
	 - FAILURE
	 - NO_SPACE
```

###Task
Task is a single node in the flow workflow. Every task can use predefined operation or subflow.

| Property | Required | Default | Description |
|----------|----------|---------|-------------|
|operation | V        |         |Describe the operation or subflow this task will run|
|publish   |          |         |Choose from the operation outputs what to publish to the flow level|
|go_to     |FAILURE: on_failure SUCCESS: go to next task| | Describe the navigation for the different results the operation has. In case of the default results, with the default go_to you don’t need to write it down.|

*sample:*

```yaml
    Task2:
          do:
            ops.compute_daylight_time_zone:
              - time_zone_as_string: flow_input
          publish:
            - daylight_time_zone
```

###Operation

Operation is the wrapper of an action.

Property    |Required    |Default	        |Description
------------|-----------|-------------------|-----------
inputs		|	        |                   |Operation inputs (see [Inputs](#docs/#inputs))
action  	|V		    |                   |Logic of the operation. (see [Action](#docs/#action))
results		|           |SUCCESS/FAILURE    |Possible results of the operation (see [Results](#docs/#results))
outputs		|   	    |                   |Outputs of the operation (see [Outputs](#docs/#outputs))

*sample:*

```yaml
    do: 
      some_operation:
        - operation_input_1: flow_input_3
        - operation_input_2
        - operation_input_3
```

###Action

Two kind of actions are supported: java @Actions and python scripts.

####@Action
A java action is a valid @Action that respects the method signature `public Map<String, String> doSomething(paramaters)` and
uses the following annotations (from `com.hp.oo.sdk.content.annotations`):

+ required annotations
    - @Param: for action parameters
+ optional annotations
    - @Action: specify action information
    - @Output: denotes action output
    - @Response: denotes action response

*sample - java @Action*

```java
    public Map<String, String> doJavaAction(
            @Param("name") String name,
            @Param("role") String role) {
        //logic here
        Map<String, String> returnValues = new HashMap<>();
        //prepare return values map
        return returnValues;
    }
```

*sample - create custom operation that uses @Action*

```yaml
    - pull_image:
            inputs:
              - imageName
              - host
              - port
              - username
              - password
            action:
              java_action:
                className: org.mypackage.MyClass
                methodName: doMyAction
            outputs:
              - returnResult
            results:
              - SUCCESS : someActionOutput == '0'
              - FAILURE
```

####Python script
You can use any traditional python 2.7 version script.

*sample - create custom operation that uses Python script*

```yaml
      - check_Weather:
          inputs:
            - city
          action:
            python_script: |
              weather = "weather thing"
              print city
          outputs:
            - weather
          results:
            - SUCCESS: 'weather == "weather thing"'
```


###Imports

Written at the beginning of the file, after the namespace, used to specify the dependencies the file is using and the name it will be referenced within the file. 
*sample:*

```yaml
   imports:
     - ops: user.ops.OpenstackOperations
     - order_vm_flow: user.flows.OrderVmFlow
     - script_file: some_script_file.py
```


###Namespace

Used at the beginning of a file. Used as the namespace of the flow or operation. 
*sample:*

```yaml
  namespace: user.flows
```

###Some more samples

####Sample1 - demonstrates custom navigation and publishing outputs

*operations:*

```yaml
    namespace: user.ops
    
    operations:
      - check_number:
          inputs:
            - number
          action:
            python_script: |
              remainder = number % 2
              isEven = remainder == 0
              tooBig = number > 512
          outputs:
            - preprocessed_number: str(fromInputs['number'] * 3)
          results:
            - EVEN: isEven == 'True' and tooBig == 'False'
            - ODD: isEven == 'False' and tooBig == 'False'
            - FAILURE # report failure if the number is too big
    
      - process_even_number:
          inputs:
            - even_number
            - offset: 32
          action:
            python_script: |
              processing_result = int(even_number) + offset
              print 'Even number processed. Result= ' + str(processing_result)
    
      - process_odd_number:
          inputs:
            - odd_number
          action:
            python_script:
              print 'Odd number processed. Result= ' + str(odd_number)
```

```yaml
    namespace: email.ops
    
    operations:
      - send_mail:
          inputs:
            - hostname
            - port
            - from
            - to
            - cc: "''"
            - bcc: "''"
            - subject
            - body
            - htmlEmail: "'true'"
            - readReceipt: "'false'"
            - attachments: "''"
            - username: "''"
            - password: "''"
            - characterSet: "'UTF-8'"
            - contentTransferEncoding: "'base64'"
            - delimiter: "''"
          action:
            java_action:
              className: org.eclipse.score.content.mail.actions.SendMailAction
              methodName: execute
          results:
            - SUCCESS: returnCode == '0'
            - FAILURE
```

*flow:*

```yaml
    namespace: user.flows
    
    imports:
     ops: user.ops
     email: email.ops
    
    flow:
      name: navigation_flow
      inputs:
        - userNumber
        - emailHost
        - emailPort
        - emailSender
        - emailRecipient
      workflow:
        check_number:
          do:
            ops.check_number:
              - number: userNumber
          publish:
            - new_number: preprocessed_number # publish the output in the flow level so it will be visible for other steps
          navigate:
            EVEN: process_even_number
            ODD: process_odd_number
            FAILURE: send_error_mail
    
        process_even_number:
          do:
            ops.process_even_number:
              - even_number: new_number
          navigate:
            SUCCESS: SUCCESS # end flow with success result
    
        process_odd_number:
          do:
            ops.process_odd_number:
              - odd_number: new_number
          navigate:
            SUCCESS: SUCCESS # end flow with success result
    
        on_failure: # you can also use this step for default navigation in failure case
          send_error_mail: # or refer it by the task name
            do:
              email.send_mail:
                - hostname: emailHost
                - port: emailPort
                - from: emailSender
                - to: emailRecipient
                - subject: "'Flow failure'"
                - body: "'Wrong number: ' + str(userNumber)"
            navigate:
              SUCCESS: FAILURE # end flow with failure result
              FAILURE: FAILURE
```

####Sample2 - demonstrates subflow usage

*operations:*

```yaml
    namespace: user.ops

    operations:
      - test_op:
          action:
            python_script: 'print "hello world"'
            
  - check_Weather:
      inputs:
        - city
      action:
        python_script: |
          weather = "weather thing"
          print city
      outputs:
        - weather: weather
      results:
        - SUCCESS: 'weather == "weather thing"'
```

*subflow:*

```yaml
    namespace: user.flows
    
    imports:
      ops: user.ops
    
    flow:
      name: child_flow
      inputs:
        - input1: "'value'"
      workflow:
        CheckWeather:
          do:
            ops.test_op:
      outputs:
        - val_output: fromInputs['input1']
```

*parent flow:*

```yaml
    namespace: user.flows
    
    imports:
      ops: user.ops
      flows: user.flows
    
    flow:
      name: parent_flow
      inputs:
        - input1
        - city:
            required: false
      workflow:
        Task1:
          do:
            ops.check_Weather:
              - city: city if city is not None else input1
          publish:
            - kuku: weather
    
        Task2:
          do:
            flows.child_flow:
              - input1: kuku
          publish:
            - val_output
      results:
        - SUCCESS
        - FAILURE
```

##SLANG Events

SLANG uses score events. An event is represented by event type (a String value) and event data (Serializable object).
In case of SLANG the event data is a map that contains all the relevant information under certain keys defined in 
`com.hp.score.lang.runtime.events.LanguageEventData` class (See [table](#docs/#event_summary) below).
SLANG extends the traditional event type set provided by score with its own event types.

Event types from score:

+ SCORE_FINISHED_EVENT
+ SCORE_ERROR_EVENT
+ SCORE_FAILURE_EVENT

Event types from SLANG and the data each provide:<a name="event_summary"></a>

+ in square brackets we provide the keys under the information is put in the event data map

| Type [TYPE] | Usage | Description [DESCRIPTION] | Timestamp [TIMESTAMP] | Execution id [EXECUTIONID] | Path [PATH] | Exception [EXCEPTION] | Call arguments [CALL_ARGUMENTS] | Special data |
|----------|----------|----------|---------|-------------|----------|----------|---------|-------------|
| EVENT_INPUT_END | Input binding finished for task |  V  |  V  |  V  |  V  |    |    | bound inputs [BOUND_INPUTS], level: task, node name [TASK_NAME]  |
| EVENT_INPUT_END | Input binding finished for operation |  V  |  V  |  V  |  V  |    |    | bound inputs [BOUND_INPUTS], level: executable, node name [EXECUTABLE_NAME]  |
| EVENT_OUTPUT_START | Output binding started for task |  V  |  V  |  V  |  V  |    |    | task publish values [taskPublishValues], task navigation values [taskNavigationValues], operation return values [operationReturnValues], level: task, node name [TASK_NAME] |
| EVENT_OUTPUT_START | Output binding started for operation |  V  |  V  |  V  |  V  |    |    | executable outputs [executableOutputs], executable results [executableResults], action return values [actionReturnValues], level: executable, node name [EXECUTABLE_NAME] |
| EVENT_OUTPUT_END | Output binding finished for task |  V  |  V  |  V  |  V  |    |    | task bound outputs [OUTPUTS], task result [RESULT], next step position [nextPosition], level: task, node name [TASK_NAME] |
| EVENT_OUTPUT_END | Output binding finished for operation |  V  |  V  |  V  |  V  |    |    | executable bound outputs [OUTPUTS], executable result [RESULT], level: executable, node name [EXECUTABLE_NAME] |
| EVENT_EXECUTION_FINISHED | Execution finished running (in case of subflow) |  V  |  V  |  V  |  V  |    |    | executable bound outputs [OUTPUTS] , executable result [RESULT], level: executable, node name [EXECUTABLE_NAME] |
| EVENT_ACTION_START | Fired before the action invocation |  V  |  V  |  V  |  V  |    |  V  | action type (Java / Python) in description |
| EVENT_ACTION_END | Fired after a successful action invocation |  V  |  V  |  V  |  V  |    |    | action return values [RETURN_VALUES] |
| EVENT_ACTION_ERROR | Fired in case of exception in action execution |  V  |  V  |  V  |  V  |  V  |    |    |    |    |    |    |
| SLANG_EXECUTION_EXCEPTION | Fired in case of exception in the previous step |  V  |  V  |  V  |  V  |  V  |    |    |


##SLANG CLI
You have two ways to obtain SLANG CLI: either download it directly from score website or build it by yourself.

###Running CLI by downloading it

+ go to ([Score website](/#/))
+ in the "Getting started section" click "Download an use slang CLI tool"
+ click "Download latest version". This will download an archive with the latest CLI version
+ unzip the archive. It contains a folder called "appassembler" and some sample flows
    - the "appassembler" folder contains the CLI tool and the necessary dependencies
+ navigate to the folder `appassembler\bin\`
+ start CLI by running `slang.bat`

###Running CLI by building it

+ download the project sources from [here](#DOWNLOAD_LINK_HERE)
+ navigate to project root directory
+ build the project: open a command window here and run `mvn clean install`
+ after building the project navigate to `score-language\score-lang-cli\target\appassembler\bin` folder
+ start CLI by running `slang.bat`

###Using the CLI

You can get a list of available commands by typing `help` in the cli console.

*sample - running a flow*

```bash
run --f c:\...\your_flow.sl --D input1=root,input2=25
```

*sample - set execution mode to asynchronous (by default the execution mode is synchronous - that means you can run only one flow at a time)*

```bash
env --setAsync true
```

*sample - get flow inputs*

```bash
inputs --f c:\...\your_flow.sl
```

*sample - slang version*

```bash
slang -version
```

The execution log is saved in `score-language\score-lang-cli\target\appassembler\bin` directory under `execution.log` name. All the events are saved in this log so using this file you can easily track your flow execution.

##Sublime integration

In order to write flows or operations in slang, you can use Sublime text editor. We provide a snippet that enables
slang templates. The files should have the .yaml , .yl or .sl extension. See below how to install / configure Sublime for Windows:

+ download and install Sublime editor from [here](http://www.sublimetext.com/2)
+ download Slang.sublime-package file from [here](https://github.com/orius123/slang-sublime)
+ copy the downloaded package file in C:\Users\<User>\AppData\Roaming\Sublime Text 2\Installed Packages

In order to use the templates start typing the template name and press enter when it appears on the screen. See below
the template types:

  Keyword  |  Description
------------|------------
  slang  |  template for a slang file
  flow  |  template for a flow
  task  |  template for a task
  operation  | template for an operation
  
*Note*: DSL elements that are not required are marked in the templates as comments (start with # sign).