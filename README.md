# pydantic-to-typescript

[![PyPI version](https://badge.fury.io/py/pydantic-to-typescript.svg)](https://badge.fury.io/py/pydantic-to-typescript)
[![CI/CD](https://github.com/phillipdupuis/pydantic-to-typescript/actions/workflows/cicd.yml/badge.svg)](https://github.com/phillipdupuis/pydantic-to-typescript/actions/workflows/cicd.yml)
[![Coverage Status](https://coveralls.io/repos/github/phillipdupuis/pydantic-to-typescript/badge.svg?branch=master)](https://coveralls.io/github/phillipdupuis/pydantic-to-typescript?branch=master)

A simple CLI tool for converting pydantic models into typescript interfaces.
It supports all versions of pydantic, with polyfills for older versions to ensure that the resulting typescript definitions are stable and accurate.

Useful for any scenario in which python and javascript applications are interacting, since it allows you to have a single source of truth for type definitions.

This tool requires that you have the lovely json2ts CLI utility installed.
Instructions can be found here: https://www.npmjs.com/package/json-schema-to-typescript

## Installation

```bash
pip install pydantic-to-typescript
```

## Pydantic V2 support

If you are encountering issues with `pydantic>2`, it is most likely because you're using an old version of `pydantic-to-typescript`.
Run `pip install 'pydantic-to-typescript>2'` and/or add `pydantic-to-typescript>=2` to your project requirements.

## CI/CD

You can now use `pydantic-to-typescript` to automatically validate and/or update typescript definitions as part of your CI/CD pipeline.

The github action can be found here: https://github.com/marketplace/actions/pydantic-to-typescript.
The available inputs are documented here: https://github.com/phillipdupuis/pydantic-to-typescript/blob/master/action.yml.

## CLI

| Prop                                          | Description                                                                                                                                                                                                                             |
| :-------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| &#8209;&#8209;module                          | name or filepath of the python module you would like to convert. All the pydantic models within it will be converted to typescript interfaces. Discoverable submodules will also be checked.                                            |
| &#8209;&#8209;output                          | name of the file the typescript definitions should be written to. Ex: './frontend/apiTypes.ts'                                                                                                                                          |
| &#8209;&#8209;exclude                         | name of a pydantic model which should be omitted from the resulting typescript definitions. This option can be defined multiple times, ex: `--exclude Foo --exclude Bar` to exclude both the Foo and Bar models from the output.        |
| &#8209;&#8209;json2ts&#8209;cmd               | optional, the command used to invoke json2ts. The default is 'json2ts'. Specify this if you have it installed locally (ex: 'yarn json2ts') or if the exact path to the executable is required (ex: /myproject/node_modules/bin/json2ts) |
| &#8209;&#8209;all&#8209;fields&#8209;required | optional. Treats all fields (even those with defaults) as required in the generated TypeScript interfaces. (Pydantic v2 only)                                                                                                           |

---

## Usage

Define your pydantic models (ex: /backend/api.py):

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List, Optional

api = FastAPI()

class LoginCredentials(BaseModel):
    username: str
    password: str

class Profile(BaseModel):
    username: str
    age: Optional[int]
    hobbies: List[str]

class LoginResponseData(BaseModel):
    token: str
    profile: Profile

@api.post('/login/', response_model=LoginResponseData)
def login(body: LoginCredentials):
    profile = Profile(**body.dict(), age=72, hobbies=['cats'])
    return LoginResponseData(token='very-secure', profile=profile)
```

Execute the command for converting these models into typescript definitions, via:

```bash
pydantic2ts --module backend.api --output ./frontend/apiTypes.ts
```

or:

```bash
pydantic2ts --module ./backend/api.py --output ./frontend/apiTypes.ts
```

or:

```python
from pydantic2ts import generate_typescript_defs

generate_typescript_defs("backend.api", "./frontend/apiTypes.ts")
```

The models are now defined in typescript...

```ts
/* tslint:disable */
/**
/* This file was automatically generated from pydantic models by running pydantic2ts.
/* Do not modify it by hand - just update the pydantic models and then re-run the script
*/

export interface LoginCredentials {
  username: string;
  password: string;
}
export interface LoginResponseData {
  token: string;
  profile: Profile;
}
export interface Profile {
  username: string;
  age?: number;
  hobbies: string[];
}
```

...and can be used in your typescript code with complete confidence.

```ts
import { LoginCredentials, LoginResponseData } from "./apiTypes.ts";

async function login(
  credentials: LoginCredentials,
  resolve: (data: LoginResponseData) => void,
  reject: (error: string) => void
) {
  try {
    const response: Response = await fetch("/login/", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(credentials),
    });
    const data: LoginResponseData = await response.json();
    resolve(data);
  } catch (error) {
    reject(error.message);
  }
}
```

### Treating all fields as required

If you are using pydantic v2 and would like to treat all fields as required in the generated TypeScript interfaces, you can use the `--all-fields-required` flag.

This is useful if you know that all fields will be present on the TypeScript side; for example, when representing a response from your Python backend API (since Pydantic will populate any missing fields with defaults before the response is sent to the client).

#### Example (pydantic v2)

```python
from pydantic import BaseModel, Field
from typing import Annotated, Literal, Optional

class ExampleModel(BaseModel):
    a: Annotated[int, Field(default=2)]
    b: Annotated[list[int], Field(default_factory=list)]
    c: Literal["c"] = "c"
    d: int = 1
    e: Optional[int]
    f: Optional[int] = None
    g: Optional[int] = 3
```

Executing with `--all-fields-required`:

```bash
pydantic2ts --module backend.api --output ./frontend/apiTypes.ts --all-fields-required
```

Generated TypeScript interface:

```ts
export interface ExampleModel {
  a: number;
  b: number[];
  c: "c";
  d: number;
  e: number | null;
  f: number | null;
  g: number | null;
}
```

Executing without `--all-fields-required`:

```bash
pydantic2ts --module backend.api --output ./frontend/apiTypes.ts
```

Generated TypeScript interface:

```ts
export interface ExampleModel {
  a?: number;
  b?: number[];
  c?: "c";
  d?: number;
  e: number | null;
  f?: number | null;
  g?: number | null;
}
```

> [!NOTE]
> Field `e` is required (not marked as optional) in the generated interface, even without the `--all-fields-required` flag. This is because, in Pydantic v2, fields annotated as `Optional[...]` or `Any` are no longer given an implicit default of `None`. See [Pydantic docs](https://docs.pydantic.dev/latest/concepts/models/#required-fields):
> > [in Pydantic V2] there are no longer any type annotations that will result in a field having an implicit default value.
