# APMN YAML Schema

JSON Schema (in YAML format) for validating APMN workflow documents.

**Namespace:** `http://apmn.kshetra.studio/ns/1.0`  
**Download:** [apmn-schema.yaml](https://raw.githubusercontent.com/kshetra-studio/apmn/main/schema/apmn-schema.yaml)

## Usage

```bash
# Validate with ajv-cli
npm install -g ajv-cli ajv-formats
ajv validate -s schema/apmn-schema.yaml -d my-workflow.apmn.yaml --spec=draft2020
```

## Schema Source

```yaml
--8<-- "schema/apmn-schema.yaml"
```
