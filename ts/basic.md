# TypeScript Core Concepts - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [TypeScript কি এবং কেন?](#typescript-কি-এবং-কেন)
- [Basic Types (মৌলিক টাইপসমূহ)](#basic-types-মৌলিক-টাইপসমূহ)
- [Interface vs Type Alias](#interface-vs-type-alias)
- [Functions (ফাংশন)](#functions-ফাংশন)
- [Classes & Access Modifiers](#classes--access-modifiers)
- [Generics (জেনেরিক্স)](#generics-জেনেরিক্স)
- [Enums (এনামস)](#enums-এনামস)
- [Union & Intersection Types](#union--intersection-types)
- [Type Assertion (টাইপ কাস্টিং)](#type-assertion-টাইপ-কাস্টিং)
- [Real-world Example](#real-world-example)

---

## TypeScript কি এবং কেন?

**TypeScript (TS)** হলো JavaScript এর একটি **Superset**। সহজ কথায়, এটি JavaScript এর সাথে **Static Typing** যুক্ত করে। ব্রাউজার সরাসরি TypeScript বুঝতে পারে না, তাই একে **Compile** করে JavaScript এ রূপান্তর করতে হয়।

### 🎯 কেন ব্যবহার করবেন?
- ✅ **Type Safety:** কোড লেখার সময়ই ভুল ধরা পড়ে।
- ✅ **Better IDE Support:** Auto-completion এবং documentation ভালো পাওয়া যায়।
- ✅ **Scalability:** বড় প্রজেক্ট মেইনটেইন করা সহজ হয়।
- ✅ **Modern Features:** ES6+ ফিচারগুলো সহজে ব্যবহার করা যায়।

---

## Basic Types (মৌলিক টাইপসমূহ)

TypeScript এ ভেরিয়েবলের টাইপ বলে দেওয়া যায়।

```typescript
// String
let userName: string = "Mahfuj";

// Number
let age: number = 25;

// Boolean
let isActive: boolean = true;

// Any (যেকোনো টাইপ - তবে এটি এড়িয়ে চলাই ভালো)
let data: any = "Hello";
data = 10; // Error দিবে না

// Array
let skills: string[] = ["JS", "TS", "React"];
let numbers: Array<number> = [1, 2, 3];

// Tuple (নির্দিষ্ট অর্ডারে ফিক্সড টাইপ)
let role: [number, string] = [1, "Admin"];
```

---

## Interface vs Type Alias

অবজেক্টের স্ট্রাকচার ডিফাইন করার জন্য `interface` এবং `type` ব্যবহার করা হয়।

### ১. Interface:
সাধারণত অবজেক্টের শেপ ডিফাইন করতে ব্যবহৃত হয় এবং এটি extend করা যায়।

```typescript
interface User {
    id: number;
    name: string;
    email?: string; // Optional property (?)
}

const user1: User = {
    id: 1,
    name: "Rahim"
};
```

### ২. Type Alias:
এটি প্রিমিটিভ, ইউনিয়ন, টিউপল ইত্যাদির জন্যও ব্যবহার করা যায়।

```typescript
type ID = number | string; // Union Type

type Product = {
    id: ID;
    title: string;
    price: number;
}

const item: Product = {
    id: "p-101",
    title: "Laptop",
    price: 50000
};
```

---

## Functions (ফাংশন)

ফাংশনের প্যারামিটার এবং রিটার্ন টাইপ ডিফাইন করা যায়।

### ১. Basic Function:
```typescript
function add(a: number, b: number): number {
    return a + b;
}

// Arrow Function
const multiply = (x: number, y: number): number => x * y;
```

### ২. Optional & Default Parameters:
```typescript
function greet(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}!`;
}

function logMessage(message: string, userId?: number): void {
    // void মানে ফাংশনটি কিছু রিটার্ন করবে না
    console.log(message, userId);
}
```

---

## Classes & Access Modifiers

OOP (Object Oriented Programming) এর কনসেপ্টগুলো TypeScript এ খুব শক্তিশালী।

### Access Modifiers:
- **public:** ক্লাসের বাইরে থেকেও এক্সেস করা যায় (Default)।
- **private:** শুধুমাত্র ক্লাসের ভেতরেই এক্সেস করা যায়।
- **protected:** ক্লাস এবং তার চাইল্ড ক্লাসে এক্সেস করা যায়।

```typescript
class Employee {
    public name: string;
    private salary: number;
    protected department: string;

    constructor(name: string, salary: number, department: string) {
        this.name = name;
        this.salary = salary;
        this.department = department;
    }

    public getSalary(): number {
        return this.salary;
    }
}

class Developer extends Employee {
    constructor(name: string, salary: number) {
        super(name, salary, "IT");
    }

    public getDept(): string {
        return this.department; // Allowed (protected)
    }
}

const dev = new Developer("Karim", 50000);
console.log(dev.name); // Allowed
// console.log(dev.salary); // Error: Private
```

---

## Generics (জেনেরিক্স)

Generics ব্যবহার করে রিইউজেবল কম্পোনেন্ট তৈরি করা যায় যা বিভিন্ন টাইপের সাথে কাজ করতে পারে। এটি `<T>` সিনট্যাক্স দিয়ে লেখা হয়।

### ১. Generic Function:
```typescript
// T হলো একটি প্লেসহোল্ডার টাইপ
function identity<T>(arg: T): T {
    return arg;
}

let output1 = identity<string>("Hello");
let output2 = identity<number>(100);
```

### ২. Generic Interface:
```typescript
interface ApiResponse<T> {
    status: number;
    data: T;
    message: string;
}

interface UserData {
    username: string;
}

const response: ApiResponse<UserData> = {
    status: 200,
    data: { username: "dev_hero" },
    message: "Success"
};
```

---

## Enums (এনামস)

Enums হলো নামযুক্ত কনস্ট্যান্টের একটি সেট। এটি কোডকে আরও রিডেবল করে।

```typescript
enum Status {
    Pending,    // 0
    InProgress, // 1
    Completed,  // 2
    Failed      // 3
}

let currentStatus: Status = Status.InProgress;

// String Enum
enum Role {
    Admin = "ADMIN",
    User = "USER",
    Guest = "GUEST"
}

if (userRole === Role.Admin) {
    // Grant access
}
```

---

## Union & Intersection Types

### ১. Union Types (`|`):
একটি ভেরিয়েবল একাধিক টাইপের হতে পারে।

```typescript
function printId(id: number | string) {
    console.log(`Your ID is: ${id}`);
}

printId(101);
printId("202");
```

### ২. Intersection Types (`&`):
একাধিক টাইপকে একত্রিত করে একটি নতুন টাইপ তৈরি করে।

```typescript
interface BusinessPartner {
    name: string;
    credit: number;
}

interface Identity {
    id: number;
    email: string;
}

type Employee = BusinessPartner & Identity;

const emp: Employee = {
    id: 1,
    name: "John",
    email: "john@example.com",
    credit: 1000
};
```

---

## Type Assertion (টাইপ কাস্টিং)

কখনও কখনও আপনি কম্পাইলারের চেয়ে ভালো জানেন যে একটি ভেরিয়েবলের টাইপ কী হবে। তখন `as` ব্যবহার করা হয়।

```typescript
let someValue: any = "This is a string";

// আমরা জানি এটি স্ট্রিং, তাই স্ট্রিং মেথড পেতে কাস্ট করছি
let strLength: number = (someValue as string).length;

// DOM Element এর ক্ষেত্রে
const input = document.getElementById("user-input") as HTMLInputElement;
console.log(input.value);
```

---

## Real-world Example

একটি ছোট **Todo Application** এর টাইপ ডেফিনিশন কেমন হতে পারে:

```typescript
// 1. Define Types
interface Todo {
    id: number;
    title: string;
    completed: boolean;
}

type TodoList = Todo[];

// 2. Class Implementation
class TodoManager {
    private todos: TodoList = [];

    public addTodo(title: string): void {
        const newTodo: Todo = {
            id: Date.now(),
            title,
            completed: false
        };
        this.todos.push(newTodo);
    }

    public getTodos(): TodoList {
        return this.todos;
    }

    public toggleTodo(id: number): void {
        const todo = this.todos.find(t => t.id === id);
        if (todo) {
            todo.completed = !todo.completed;
        }
    }
}

// 3. Usage
const manager = new TodoManager();
manager.addTodo("Learn TypeScript");
manager.addTodo("Build a Project");

console.log(manager.getTodos());
```

---

## 🎯 Best Practices:
- ✅ সবসময় `any` এড়িয়ে চলুন, প্রয়োজনে `unknown` ব্যবহার করুন।
- ✅ অবজেক্টের জন্য `interface` ব্যবহার করুন।
- ✅ ফাংশনের রিটার্ন টাইপ সবসময় উল্লেখ করুন।
- ✅ `strict: true` মোডে `tsconfig.json` কনফিগার করুন।

এই গাইডটি অনুসরণ করলে আপনি TypeScript এর মূল বিষয়গুলো আয়ত্ত করতে পারবেন এবং যেকোনো প্রজেক্টে কনফিডেন্সের সাথে কাজ করতে পারবেন।