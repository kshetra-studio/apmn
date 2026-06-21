# APMN BPMN Extension XSD

XML Schema for using APMN extensions inside standard BPMN 2.0 `extensionElements`.

**Namespace:** `http://apmn.kshetra.studio/ns/1.0`  
**Download:** [apmn-extension.xsd](https://raw.githubusercontent.com/kshetra-studio/apmn/main/schema/apmn-extension.xsd)

## Usage in BPMN 2.0

```xml
<definitions xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
             xmlns:apmn="http://apmn.kshetra.studio/ns/1.0">
  <process id="my_process">
    <task id="task_1" name="Classify document">
      <extensionElements>
        <apmn:agentTask
          model="gemini-2.0-flash"
          system_prompt="Classify the document type and extract key fields."
          confidence_threshold="0.85"/>
      </extensionElements>
    </task>
  </process>
</definitions>
```

## Schema Source

```xml
--8<-- "schema/apmn-extension.xsd"
```
