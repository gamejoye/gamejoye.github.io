---
title: es5-impl-class
date: 2024-10-03 12:55:22
tags: 前端
cover: cover.jpeg
---

```
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function() {
  console.log(this.name + ' makes a noise.');
};
function Dog(name, breed) {
  Animal.call(this, name);
  this.breed = breed;
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.bark = function() {
  console.log(this.name + ' barks.');
};

var dog = new Dog('Rex', 'Labrador');
dog.speak(); // 输出: Rex makes a noise.
dog.bark();  // 输出: Rex barks.
```

