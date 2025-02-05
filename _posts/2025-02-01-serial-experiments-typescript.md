---

title: Getting code execution with TypeScript reflection

---
<img src="../assets/img/typescript-viewing-reflection.webp" width="100%">
# Intro

Unfortunately, we live in a world where memory exploitation is a less and less useful art. The next best thing in my opinion, is reflection; the cool hacking of object oriented programming languages. Reflection allows one to instantiate and manipulate types/classes during runtime and can allow for funky behavior, including of course RCE.

All deserialization and reflection vulnerabilities rely on a class which does *something* dangerous when it gets instantiated. We will be deep diving into typescript's reflection capabilities. Though, for ease of testing, we will use the following type (called **XXX**) as the dangerous class we want to attempt to instantiate and abuse through reflection/deserialization. When XXX has it's **Data** property set it will run whatever string is supplied as a system command.

```typescript
class XXX {

    private _data: string = "";
    public get Data(): string {
        return this._data;
    }
    public set Data(value: string) {
        this._data = value;
        this.executeCommand(value);
    }

    private executeCommand(command: string): void {

        exec(command, (error, stdout, stderr) => {

            if (error) {

                console.error(`exec error: ${error}`);

                return;

            }

            console.log(`stdout: ${stdout}`);

            console.error(`stderr: ${stderr}`);

        });

    }

}
```

---
# Going from string -> type

When I first learned about reflection and deserialization vulnerabilities, a mentor described the "original sin" of reflection. Which is, getting a type (or constructor) from a string. So going from just a simple string "XXX" to the type XXX. This is considered the "original sin" of reflection. Let's look at the ways in which we can accomplish this in typescript/javascript.
## Eval

```typescript
class XXX {
	...
}

let userInput : string = "XXX";
eval(userInput);
// will output the constructor of XXX
```
In the above code we see a class name passed to the eval function. When a class name is passed to eval, what gets returned is the constructor of the specified class. This is the first way to go from a string to a type. But, the problem is eval is known to be a very dangerous function and will having user input flow to eval typically will mean you have much bigger problems. 

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
In the above code block we see user input flowing into an await import statement. This gets whatever classes are exported from the specified js file. Within XXX.ts (which will be compiled to XXX.js) we export the class XXX. 

## SystemJS

A third party npm library which offer a faster alternative for dynamic importing. 
```typescript
System.import("module-to-import-here")
```

---
# Basic TypeScript/JavaScript Reflection

```typescript
async function main()

{
    let userInput = "XXX";
    const sometype = await dynamicImport(userInput);
    // instantiate class with object.create()
    const obj : object = Object.create(sometype.prototype);  
    obj["Data"] = "whoami"; // trigger the RCE
}
main()
.then(res=>{console.log("done!")});
```
The first most primitive way to instantiate a class with user input would be using the **Object.create()** function.

---
# Reflect API

```typescript
// Instantiate object of type 'XXX'
const obj : object = Reflect.construct(eval("XXX"), []);

// Use Reflect.set to set the value of 'Data'
const propName = "Data";

const success = Reflect.set(obj, propName, "whoami");
```
In this code block, we instantiate a class of type **XXX** and then set it's **Data** field to be value **whomai**. Notice though, we rely heavily on the "eval" statement. We need to go from the string "XXX" to the class XXX.

---
# Class-Transformer library
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

## Digging into plainToClass Implementation

We know plainToClass is invoked with a constructor passed to it, which is of the **function** type. Then this function is passed to the TransformExecutorOperation.js method **transform()**. Eventually our provided constructor is executed like this.

```javascript
		...
        } else if (targetType) {
          newValue = new (targetType as any)(); // this instantiates our constructor
        } else {
	    ...
        }
```

This technique to call the provided constructor function, is known as **dynamic instantiation**.

---
# Github Hunting

## nodomjs/nodomts
```typescript
public static deserialize(jsonStr:string):Element{
	let jObj = JSON.parse(jsonStr);
	...
	if(jsonObj.hasOwnProperty('className')){
					const cls:string = jsonObj['className'];
					let param = [];
					//指令需要传入参数
					switch(cls){
						case 'Directive':
							param = [jsonObj['type']];
							break;
						case 'Expression':
							param = [jsonObj['execString']];
							break;
						case 'Element':
							param = [];
							break;
						case 'NodomEvent':
							param = [jsonObj['name']];
							break;
					}
					let clazz:any = eval(cls);
					retObj = Reflect.construct(clazz,param);
				}else{
					retObj = {};
				}
	...
}
```

## carbon-io/atom