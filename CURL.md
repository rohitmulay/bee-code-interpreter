### **Test: Imports**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"source_code": '"$(jq -Rsa . < ./examples/using_imports.py)"', "files": {}}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute
```

### **Test: Ad-Hoc Import**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"source_code": '"$(jq -Rsa . < ./examples/cowsay.py)"', "files": {}}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute
```

### **Test: Create File in Interpreter**
1. **Step 1: Create the File**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"source_code": "with open(\"file.txt\", \"w\") as f:\n    f.write(\"Hello, World!\")", "files": {}}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute
```

2. **Step 2: Read the File**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"source_code": "with open(\"file.txt\", \"r\") as f:\n    print(f.read())", "files": {"/workspace/file.txt": "<file_content_base64>"} }' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute
```
> Replace `<file_content_base64>` with the actual base64 content of the file returned in the previous response.

### **Test: Parse Custom Tool (Success)**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"tool_source_code": "def my_tool(a: int, b: typing.Tuple[Optional[str], str] = (\"hello\", \"world\"), *, c: typing.Union[list[str], dict[str, typing.Optional[float]]]) -> int:\n    \"\"\"\n    This tool is really really cool.\n    Very toolish experience:\n    - Toolable.\n    - Toolastic.\n    - Toolicious.\n    :param a: something cool\n    (very cool indeed)\n    :param b: something nice\n    :return: something great\n    :param c: something awful\n    \"\"\"\n    return 1 + 1"}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/parse-custom-tool
```

### **Test: Parse Custom Tool (Success 2)**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"tool_source_code": "import typing\nimport requests\n\ndef current_weather(lat: float, lon: float):\n    \"\"\"\n    Get the current weather at a location.\n\n    :param lat: A latitude.\n    :param lon: A longitude.\n    :return: A dictionary with the current weather.\n    \"\"\"\n    url = \"https://fake-api.com/weather?lat=\" + str(lat) + \"&lon=\" + str(lon)\n    response = requests.get(url)\n    response.raise_for_status()\n    return response.json()"}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/parse-custom-tool
```

### **Test: Execute Custom Tool (Success)**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"tool_source_code": "def adding_tool(a: int, b: int) -> int:\n  return a + b", "tool_input_json": "{\"a\": 1, \"b\": 2}"}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute-custom-tool
```

### **Test: Parse Custom Tool (Error)**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"tool_source_code": "def my_tool(a, /, b, *args, **kwargs) -> int:\n  return 1 + 1"}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/parse-custom-tool
```

### **Test: Execute Custom Tool (Error)**
```bash
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"tool_source_code": '"$(echo 'def division_tool(a: int, b: int) -> int:
  return a / b' | jq -Rs .)"', "tool_input_json": '"$(echo '{"a": 0, "b": 0}' | jq -Rs .)"'}' \
     http://code-interpreter-route-bee-code-interpreter.apps.wo-510-sb-ng-5.cp.fyre.ibm.com/execute-custom-tool

```
