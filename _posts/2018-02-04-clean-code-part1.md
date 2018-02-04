---
layout: post
title: Clean Code - why should we care and using meaningful names
header-img: "img/post-bg-06.jpg"
comments: true
---

### Chapter 1: Why should we care about Clean Code
Its been around 2 years since I started programming professionally. During this time I have worked with several languages and on several projects. Even though I can learn new things fast and switch between languages as per the requirement (thanks to Google, StackOverflow and all the awesome bloggers), I felt I had more to learn about writing understandable and elegant code. I thought maybe it would get better with more practice and experience. But one of the recent code review discussions led me to think **"what is the exact definition of "good" code?"** and **"is there any specification defining how to write good code?"**. I did some search and stumbled upon an excellent book **Clean Code** by **Robert Martin** or **uncle Bob** as we know him. This was just what I needed at that moment.

After reading a couple of chapters, I thought it would be helpful to maintain concise notes highlighting the important points I get while reading this book. So my motivation behind this post is to share my learnings about writing cleaner and better code. All the points and ideas discussed below is my understanding of chapters 1 and 2 of the above mentioned book. If there's any suggestion or improvements feel free to reach out.

Getting to the topic of this blog post: **why should we care about clean code?**. Thats a great question. Someone may ask why even bother spending extra time and efforts on improving or maintaining the existing code when the end user can only see the final product and not the code itself. We could use this time to build new things instead of improving the existing code (as long as it works), right?

Lets explore the idea behind writing good code or **clean code**. But to write clean code, we have to understand what actually is clean code. To answer that, we'll start from the issue and move towards how clean code can help us solve it.

1. **Bad code slows us down.** Martin explains that the productivity of development team decreases with the passage of time, eventually reaching zero. We (the developers) let bad code to degrade until the point when it is not possible to work on it. At this point a major refactor would be required.

2. **Attitude.** Accepting that the mess is our (programmer’s) fault and not it's due to manager, deadline, customers, etc. Its our responsibility not to compromise on the quality of codebase due to pressure of delivering on schedule.

3. **Acceptance and realisation.** Now we understand the downfalls of writing bad code and also the fact that the best way to meet those deadlines is by writing clean and maintainable code.

4. **The big question.** Probably at this point you will be asking *“ok, but what do you mean by clean code?”*. Thats a good question because to write a clean code we should understand the meaning of it.

5. **Finally, what is clean code?** The best answer to this question that I found in the book is by Dave Thomas:
> “Clean code can be read, and enhanced by a developer other than its original author. It has unit and acceptance tests. It has meaningful names. It provides one way rather than many ways for doing one thing. It has minimal dependencies, which are explicitly defined, and provides a clear and minimal API.” - Dave Thomas, founder of OTI.

The explanation is pretty much straight forward and covers most of the properties of a good code. In my opinion, **clean code is code which is easy to read and modify especially by other developers**


### Chapter 2: Meaningful Names

**Note:** Examples and points below are explained in terms of Java but they can be equally applicable for other languages as well.

1. **Names should reveal intention** - Choosing class, function or variable names that captures the intent behind their usage and easy to understand.
Member name should indicate clearly what it's storing, function name should clearly tell what it's doing to a point where things become obvious for others as they reading your code, without the need to document it. It takes time to get it right but it's worth it.
Referring to the below example, it is clear that using proper names can make the code much easier to read.
```java
// Not a meaningful name
int d; // elapsed time in days
// Meaningful names
int elapsedTimeInDays;
int daysSinceCreation;
int daysSinceModification;
int fileAgeInDays;
```

2. **Make meaningful distinctions** - Distinguish names in a way that there's no ambiguity between any of them. For example `customer` and `customerInfo`, `customerData` seem indistinguishable as the latter two contain *noise words* like *data* and *info* which reveal nothing significant about the variable.<br>
Also method signature `void copyChars(char source[], char destination[])` reads much better than `void copyChars(char a1[], char a2[])`.

3. **Use pronounceable names** - Doing so would make it easier to communicate with other developers and it would also make it easier for the new members to understand the code faster.<br>
For example, if you have a variable named `modymdhms` which stores file's modification timestamp, then good luck while discussing about it with your peers. Instead `modificationTimestamp` seems much better alternative.

4. **Use searchable names** - It helps in finding occurrences of variables and constants if they are used in multiple places in a code base. It would be much easier finding usages for a constant named `ALLOWED_QUERIES_PER_SEC` than a numeric value such as `2` or variable named `i`. This makes searching much faster using the IDEs.

5. **Don’t use type encodings** - Avoid embedding type of scope related information in names such as member prefix (or Hungarian notation used in for example Android Open source style guidelines)  e.g. `mDeliveryAddress`. They add unnecessary burden and are difficult to read and pronounce. Plus syntax highlighting and other features of modern IDEs render these practises not so useful.<br>
Don't embed types into members names like `phoneString` denoting a string type especially with strongly typed languages like Java, Go, etc. Just `phone` would be better.<br>
Also prefixing **I** with an interface name like `IPostRepository` should be avoided. Instead name the interface as `PostRepository` and its implementations like `MongoDBPostRepository` or even `PostRepostioryImpl`.

6. **Avoid mental mapping** - Reader of the code shouldn't have to keep mental mapping of variable names to something else. For example using a single character variable name like `i` or `j` is ok if it exists within a very small scope such a a simple `for` loop, but is problematic while working on a 3-level nested `for` loop. The mental mapping maybe an indication of poor variable naming.

7. **Class and variable names** - class names should be noun or noun phrases like `Customer`, `Accounts` or `AddressParser`. Avoid class name which have `Manager`, `Data` or `Info` in them. Method names should be a verb or verb phrase like `deletePost` or `save`.

8. **Choose one word per concept** - For example using words with similar intent like `fetch`, `retrieve` or `get` in the codebase get confusing and bit unpredictable to recall names. It would be same for class names: `AccountsManager` or `AccountsController`. Instead choose one of them and use it consistently.


Hopefully I was able to communicate the need to write good code and it starts with proper naming. Hopefully, I will add new posts about clean code as I make progress reading the book.

> "In short, a programmer who writes clean code is an artist who can take a blank screen through a series of transformations until it is an elegantly coded system." - Robert Martin.
