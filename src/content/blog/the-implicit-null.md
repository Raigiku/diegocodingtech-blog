---
title: 'The Implicit NULL'
description: 'Lorem ipsum dolor sit amet'
pubDate: 'Oct 21 2023'
heroImage: '/the-implicit-null.jpg'
---

Computers follow human instructions, making it crucial to express our ideas explicitly in code to prevent bugs. Unfortunately, for many years mainstream programming languages have fallen short in achieving this precision, primarily due to one feature: the implicit null.

When I mention the implicit null, I'm encompassing one or both of the following scenarios:

1. If a variable is of type X, it can accept a null value without necessitating explicit declaration of X as nullable.

2. If a variable of type X has a null value assigned, attempting to invoke methods or access fields of type X will result in a runtime error due to the null value.

These two scenarios can be seen in the example below:
```csharp
// C# example
class Program
{
    static void Main() 
    {
        var person = new Person();
        sendEmail(person);
    }

    static void SendEmail(Person person) 
    {
        // this will fail at RUNTIME
        var formattedName = person.Name.ToUpper();
        // ...
    }
}

class Person 
{
    public string Name;
}
```

The code above will fail due to the `Name` property of the `Person` instance being null. The `ToUpper` method, which is applicable only to strings, cannot be executed in this scenario. It's crucial to note that this failure occurs at runtime, not compile time. Taking into account human oversight, this error can easily be missed, potentially resulting in problems in a production environment.

Such issues often arise when a programming language incorporates implicit nulls as a feature. An implicit null is typically allowed for referenced types, such as strings, objects, and dynamic arrays, primitive types do not normally have this allowance. Given that referenced types are widely utilized in codebases, it can be challenging to ensure that these variables are consistently non-null within software systems.

For example, let's consider string types. They are widely used in many contexts. When I look at my ID government card, I notice that every person in my country has a name, last name, and nationality—all of which are non-nullable strings. Surprisingly, for many years, even major programming languages like JavaScript, Python, Java, C#, C++, PHP, and Go have lacked the capability to comply this **basic requirement**.

In such cases, it's common to overlook the possibility of null values and proceed as though these fields are inherently non-nullable. The alternative involves checking for null values in every function that interacts with these variables, resulting in increased complexity and the addition of boilerplate code. Consequently, due to human error, it is not uncommon to encounter a null runtime exception. If our systems were designed to autonomously enforce this straightforward rule, rather than solely relying on human awareness, we could eliminate these hidden runtime exceptions and the need for explicitly checking null values for variables that are expressly defined as non-nullable by business rules.

Another example that shows the issues with implicit nulls is when you use a function from a library that needs a parameter. Depending on the parameter's value, it may return null. However, since the programming language can't represent nullable types, it's not possible to determine if a null value could be returned just by looking at the function definition. You have to go to the library's documentation and check if it mentions the possibility of a null return, praying that the documentation is well-written.

Fortunately, there is a rising trend towards null-safety that has emerged in several programming languages, such as TypeScript, Kotlin, Rust, Dart, and C# 8+. This development brings hope for a significant reduction in bugs within software systems. As these languages introduce nullable data types, which necessitate explicit definition, and they also enforce the practice of checking for non-null values before calling methods or accessing fields. Here's an example:

```ts
// TypeScript example
type Product = {
  // nullable data type
  shippingDetails: ShippingDetails | null;
};

function sendEmail(address: string) {
  // ...
}

function notifyCustomerA(product: Product) {
  // this will fail at COMPILATION time
  sendEmail(product.shippingDetails.address)
}

function notifyCustomerB(product: Product) {
  // this will not fail
  if (product.shippingDetails !== null)
    sendEmail(product.shippingDetails.address)
}
```

Put plainly, business rules dictate that software systems should employ non-nullable data types. Allowing implicit nulls is problematic as it introduces a mismatch that results in code not aligning with these rules, leading to bugs.