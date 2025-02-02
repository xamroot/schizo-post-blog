---

title: Getting code execution with TypeScript reflection

---

## Basic TypeScript Reflection

```typescript
// Instantiate object of type 'XXX'
const obj : object = Reflect.construct(eval("XXX"), []);

// Use Reflect.set to set the value of 'Data'
const propName = "Data";

const success = Reflect.set(obj, propName, "whoami");
```
In this code block, we instantiate a class of type **XXX** and then set it's **Data** field to be value **whomai**. Notice though, we rely heavily on the "eval" statement. We need to go from the string "XXX" to the class XXX. For any future reflection abusing we will need to get from a string to a constructor, so that begs the question: "is eval the only way to get from a string to a constructor"?

## Dynamic Import Statements

```typescript
// file XXX.ts
export default class XXX {
	... // some class definition here
}

...

// file main.ts
var userInput : string = "XXX";
// dynamic import
const module = await import(`./${className}.js`);
const type = module.default; // Assuming the constructor is exported as default
```
In the above code block we see user input flowing into an await import statement. This gets whatever classes are exported from the specified js file. Within XXX.ts (which will be compiled to XXX.js) we export the class XXX. This is seemingly the only other way (other than eval) to get a class definition from a string in typescript/javascript.
## Class-Transformer library
### RCE via Reflection

```typescript
import { plainToClass } from "class-transformer";
...
// instantiate class
const someobj:object = plainToClass(eval("XXX"), {});

// set the Data property on the gadget -> RCE
someobj["Data"] = "whoami"
```
This code block achieves the same functionality as the previous codeblock, without needing the reflection api.
### Serialization/Deserialization behavior
