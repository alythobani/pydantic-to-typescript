# Fork of pydantic-to-typescript

This is a personal fork of [pydantic-to-typescript](https://github.com/phillipdupuis/pydantic-to-typescript/) with a couple of small features added that I needed for my project. (The original library is great, but not very actively maintained at the time of this writing.)

## Extra Features in This Fork

- `--all-fields-required` flag: ensures all fields are treated as required in TypeScript output (when off, fields with defaults get marked as optional)
- Fix for [index signatures being added to indirectly imported models](https://github.com/alythobani/pydantic-to-typescript/issues/3) in the output

Feel free to use this fork if those features are useful to you. I don't have time to maintain it super actively myself (or even publish it); to use it myself I currently just copy/paste the code into `cli/script.py` of the installed `pydantic-to-typescript` package.

The rest of this README is copied from the original project (aside from the [Treating all fields as required](#treating-all-fields-as-required) section I've added).

---

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
| &#8209;&#8209;all&#8209;fields&#8209;required | optional (off by default). Treats all fields as required (present) in the generated TypeScript interfaces.                                                                                                                              |

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

If you would like to treat all fields as required in the generated TypeScript interfaces, you can use the `--all-fields-required` flag.

This is useful, for example, when representing a response from your Python backend API—since Pydantic will populate any missing fields with defaults before sending the response.

#### Example

```python
from pydantic import BaseModel, Field
from typing import Annotated, Literal, Optional

class ExampleModel(BaseModel):
    literal_str_with_default: Literal["c"] = "c"
    int_with_default: int = 1
    int_with_pydantic_default: Annotated[int, Field(default=2)]
    int_list_with_default_factory: Annotated[list[int], Field(default_factory=list)]
    nullable_int: Optional[int]
    nullable_int_with_default: Optional[int] = 3
    nullable_int_with_null_default: Optional[int] = None
```

Executing with `--all-fields-required`:

```bash
pydantic2ts --module backend.api --output ./frontend/apiTypes.ts --all-fields-required
```

```ts
export interface ExampleModel {
  literal_str_with_default: "c";
  int_with_default: number;
  int_with_pydantic_default: number;
  int_list_with_default_factory: number[];
  nullable_int: number | null;
  nullable_int_with_default: number | null;
  nullable_int_with_null_default: number | null;
}
```

Executing without `--all-fields-required`:

```bash
pydantic2ts --module backend.api --output ./frontend/apiTypes.ts
```

```ts
export interface ExampleModel {
  literal_str_with_default?: "c";
  int_with_default?: number;
  int_with_pydantic_default?: number;
  int_list_with_default_factory?: number[];
  nullable_int: number | null; // optional if Pydantic V1
  nullable_int_with_default?: number | null;
  nullable_int_with_null_default?: number | null;
}
```

> [!NOTE]
> If you're using Pydantic V1, `nullable_int` will also be optional (`nullable_int?: number | null`) when executing without `--all-fields-required`. See [Pydantic docs](https://docs.pydantic.dev/2.10/concepts/models/#required-fields):
> > In Pydantic V1, fields annotated with `Optional` or `Any` would be given an implicit default of `None` even if no default was explicitly specified. This behavior has changed in Pydantic V2, and there are no longer any type annotations that will result in a field having an implicit default value.
