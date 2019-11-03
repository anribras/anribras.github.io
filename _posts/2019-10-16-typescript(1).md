---
layout: post
title:
modified:
categories: Tech
tags: [web,typescript]
comments: true
---



<!-- TOC -->

- [基本概念](#基本概念)
- [交叉类型](#交叉类型)

<!-- /TOC -->

官方文档:
<https://www.tslang.cn/docs/handbook/enums.html>

playground:

<http://www.typescriptlang.org/play/>

## 基本概念

```ts
const message: string = 'hello world';
let city=  'Changsha'
let s: string = `I come ${city}`;
let anyThing: any;
anyThing = 6;
anyThing = 8;
//anyThing.callAnything();

//1 联合类型
let unionType: string | object | [];
unionType = [1, 2, 3];
unionType = 'string';
unionType = { 1: 1, 2: 2 };

interface IExpand {
    [propName:string]: any
}
//2 接口
//Uint16Array = false;
interface IStory extends IExpand {
    //只读属性
    readonly title: string;
    subTitle?: string,
    author: number | string;
    content: string,
    isPublished: boolean
}

//Uint16Array = false;
interface IExpandStory extends IStory {
    //可以扩展
    [propName:string]:any
}

//严格类型限定
interface IPerson {
    name: string,
    age: number,
    story: IStory[],
    //可有可无
    expandStory?: IExpandStory[]
}

//函数参数、返回类型限定
const f = (person: IPerson):number => {
    alert(JSON.stringify(person))
    return 0;
};

f({
    name: 'a',
    age: 1,
    story: [{
        title: '1',
        author: 1,
        content: 'nice',
        isPublished: false,
    }],
    expandStory: [{
        title: '1',
        author: "bravo",
        content: 'nice again',
        isPublished: true,
        extra: `It's fake!`
    }]
})




//函数interface定义,函数签名
interface dumySomething {
    ( person: IPerson):void
}
// 使用
const g: dumySomething = (person: IPerson) => { 
    alert(`In g: ${JSON.stringify(person)}` )
};

g({
    name: 'a',
    age: 1,
    story: [{
        title: '1',
        author: 1,
        content: 'nice',
        isPublished: false,
    }],
})

//可索引的类型
interface IStringArray {
    [index: number]: string
}

let stringA: IStringArray = ['1', '2', '3'];
//let stringB: IStringArray = ['1', '2', 3];


//类类型,用来限定类的
interface ClockInterface {
    currentTime: Date;
}

class Clock implements ClockInterface {
    currentTime: Date;
    constructor(h: number, m: number) { }
}

interface IDateTime {
    //currentTime: Date;
    setTime(date: Date): Date;
}

class DateTime implements IDateTime {

    //currentTime: Date;
    setTime(date: Date) {
        console.log(date);
        //this.currentTime = date;
        return date;
    };
    constructor(){}
}


//typescript 天生虚函数

interface IAction {
    move(): void;
}
class Animal implements IAction {
    move(): void {
        console.log('move')
    };
    constructor(){}
}

class Rabit extends Animal {
    move(): void {
        alert('Rabit:');
        super.move()
    };
    // //显式的子类contructor必须调用super调用父类的super();
    // constructor() {
    //     super();
    // }
}

class Mouse extends Animal {
    move(): void {
        alert('Mouse:');
        super.move()
    };
        // //显式的子类contructor必须调用super调用父类的super();
    constructor() {
        super();
    }
}

const a: Animal = new Rabit();
a.move()


//参数属性
class Octopus {
    readonly numberOfLegs: number = 8;
    //参数readonly 相当于增加了1个readonly的只读属性,更简便
     //readonly name: string
    constructor(readonly name: string) {
    }
}

const o = new Octopus('121');
alert(o.name);


//这够不够看
let someF: (a: number | string, b: object[]) => number
    = (a: number | string, b: object[]) => {
    return 0;
    }

//类似函数重载,只是用定义来明确
function fun(a: number): void ; 
function fun(a: string): number;
//实现
function fun(a: any | any[] | {} ): any { 
    if (typeof (a) === 'string') {
        alert(`string: ${a}`)
        return 0;
    } else if  (typeof(a) === 'number') {
        alert(`number: ${a}`)
    }
 }



fun(1);
fun('1');
// 就是为了这个报错，不是重载模板定义的东西，直接报错，
//因为不能分开定义，比c++还是弱了点
//fun({});


//泛型
interface Args<T> {
    a: T,
    b: T[]
}
//这个很c++的模板神似
function templateFunc<T>(a: T, b: T[], args: Args<T>): number { return 0; }

//自动推导就完了 
let templateFuncInstance = templateFunc;
//显式把函数签名
let templateFuncInstance1: {<T>(a:T,b:T[],args:Args<T>):number} = templateFunc;

//泛型工厂，强制第1个参数有new的实现，也就是class,或者构造函数
function create<T>(c: {new(s:string): T; }, ss:string="default"): T {
    return new c(ss);
}

class Strings {
    _s: string;
    constructor(s: string) {
        this._s = s;
    };
    dump():void  { 
        alert(this._s);
    }
}

class Love {
    _s: string;
    constructor(s: string) {
        this._s = s;
    };
    dump():void  { 
        alert(this._s);
    }
}

//工厂函数
create<Strings>(Strings).dump();
create<Strings>(Strings, 'string').dump();
create<Strings>(Love, 'love').dump();


//枚举

enum Color { red=1, green };
let c: Color = Color.red;

```

## 交叉类型

这个`T & U`叫做交叉类型

混入mixin的实现:

```ts
function extend<T, U>(first: T, second: U): T & U {
    let result = <T & U>{};
    for (let id in first) {
        (<any>result)[id] = (<any>first)[id];
    }
    for (let id in second) {
        if (!result.hasOwnProperty(id)) {
            (<any>result)[id] = (<any>second)[id];
        }
    }
    return result;
}

class Person {
    constructor(public name: string) { }
}
interface Loggable {
    log(): void;
}
class ConsoleLogger implements Loggable {
    log() {
        // ...
    }
}
var jim = extend(new Person("Jim"), new ConsoleLogger());
var n = jim.name;
jim.log();
```
