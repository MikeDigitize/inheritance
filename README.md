# Inheritance
Experiments with inheritance patterns in JavaScript.

The motivation behind this was I wanted to create a wrapper around an API that needed some love / syntactic sugar. There was lots of stuff that I could do with this API. First, just make it easier and friendlier to use. This would be like a base class - a lightweight convenient wrapper around the API that would let people use it without extra features they didn't need. 

There were other additional features I could build into the base class but I wanted these to be optional, housed in separate files and importable on demand. So I thought I'd investigate how to do this using traditional JavaScript inheritance.

```javascript
// base-class.js

function Person(name) {
  this.name = name;
}

// employee-class.js

function Employee(name, occupation) {
  this.occupation = occupation;
  Person.call(this, name);
}

// custom.js

var me = new Employee('Mike', 'developer');

// gender-class.js

function Gender(name, gender) {
  this.gender = gender;
  Person.call(this, name);
}

// custom.js

var me = new Gender('Mike', 'male');

```

So as long as my build process is designed in a way that allows me to bundle the base class with extensions I can pick and choose to add whatever functionality I needed. For example, with Gulp:

```javascript
gulp.task('js:base-employee', () => {
    return gulp.src(['./src/js/base-class.js', './src/js/employee-class.js'])
        .pipe(concat('base-class-employee.js'))
        .pipe(gulp.dest(jsDest));
});

```
Ok, cool but things to note from this strategy are that the additional functionality requires me to change the name of the base class, I'm limited to a single extension at a time, I have to use the base class within each extension's constructor, it needs the `new` keyword to create instances and it doesn't use any inheritance tools that came with ES5 or ES6.

Ok I'll try that again without changing the base class.

```javascript

// base-class.js

function Person(name) {
  this.name = name;
}

// employee-class.js

Person.prototype.setOccupation = function(occupation) {
	this.occupation = occupation;
};

// gender-class.js

Person.prototype.setGender = function(gender) {
	this.gender = gender;
};

// custom.js

var me = new Person('Mike');
me.setOccupation('developer');
me.setGender('male');

```

Ok so now I'm keeping the base class name intact but now I'm having to create methods on the base class prototype to set additional properties. So is it worth trying something unconventional with the base class?

```javascript

// base-class.js

function Person(name, extend) {
	this.name = name;
	if(extend) {
		extend.forEach(fn => fn.constructor.call(this, fn.prop));
	}	
}

// employee-class.js

function Employee(occupation) {
  this.occupation = occupation;
}

// gender-class.js

function Gender(gender) {
	this.gender = gender;
}

// custom.js

var mike = new Person('Mike', [
	{ constructor : Employee, prop : 'developer'},
	{ constructor : Gender, prop : 'male'}
]);

```

So in the above the base class is modified to take an additional argument - an array of objects with a constructor property and initialisation property that will internally get set against the base class. But it's not a nice API. It's also inflexible. Maybe I could use `apply` instead of `call` and pass in an array of initialising properties if there's a need for more than one. Still it's far from ideal. Besides, with such arbitrary extensions it's hard to ascertain any practical use for the above so I'll change the base class to something slightly more practical.

```javascript

// base-class.js

function DomTool(selector, extend) {
	this.elements = Array.from(document.querySelectorAll(selector));
	if(extend) {
		extend.forEach(fn => fn.constructor.call(this, fn.prop));
	}	
}

// onclick-class.js

function Click() {
  if(!this.callbacks) {
    this.callbacks = [];
  }
  this.addClick = function(callback) {
	  this.callbacks.push(callback);
	  this.elements.forEach(element => element.addEventListener('click', callback));
	}
}


// style-class.js

function Style() {
	this.addStyles = function(style) {
	  var key = Object.keys(style)[0];
	  this.elements.forEach(element => element.style[key] = style[key]);
	}
}

// custom.js

var h1 = new DomTool('h1', [
	{ constructor : Click },
	{ constructor : Style }
]);

```

So now via the above I can extend the base class with extensions. But these extensions all reference the `elements` property from the base class so are useless without the base class. This isn't such a bad thing when designing a specific extensible API. If I were bothered about that I could just use composition instead.


```javascript

// base-class.js

function DomElement(selector) {
  this.elements = Array.from(document.querySelectorAll(selector));
}

// onclick-class.js

function addClick(callback, elements) {
  if(!elements) {
    elements = this.elements;
  }
  if(!Array.isArray(elements)) {
    elements = [elements];
  }
  elements.forEach(element => element.addEventListener('click', callback));
}


// style-class.js

function addStyle(style, elements) {
  if(!elements) {
    elements = this.elements;
  }
  if(!Array.isArray(elements)) {
    elements = [elements];
  }
  var key = Object.keys(style)[0];
  elements.forEach(element => element.style[key] = style[key]);
}

// custom.js

DomElement.prototype.addClick = function(callback) {
  addClick.call(this, callback);
};

DomElement.prototype.addStyle = function(style) {
  addStyle.call(this, style);
}

var h1 = new DomElement('h1');

```

So now with a bit of internal testing of arguments I have a series of functions that can be composed as part of a base class or can be used on their own. `DomElement` is not exactly a particularly great example of a base class but it provides the element that the other functions can act upon so it's ok I guess. But I'm still having to use the `new` keyword to instantiate an instance. I'll try the same thing again using `Object.create`.

```javascript

// custom.js

var DomTool = function(element) {
	return Object.create(DomElement.prototype, {
		elements : {
			writable: true, 
		  	configurable: true, 
		  	value: Array.from(document.querySelectorAll(element))
		},
	  	onClick: { 
		  	writable: false, 
		  	configurable: true, 
		  	value: function(callback) {
			    addClick.call(this, callback);
			}
		},
	  	addStyle: {
		    writable: false, 
		  	configurable: true, 
		  	value: function(style) {
			    addStyle.call(this, style);
			}
		}
	});
}

var h1 = DomTool('h1');

```

So now I've eliminated the need to use `new` but there's a fair amount of setup required in our custom.js file to compose the functions together. And I'm not using inheritance per se I've created a factory that spits out an object. The instance inherits from DomElement not from DomTool. 

This is one way to do it but it's not the most ideal, is rather verbose and lacks sophistication. So could ES6 classes offer any benefits that other methods explored haven't?

```javascript

// base-class.js

export class DomElement {
  constructor(selector) {
    this.elements = Array.from(document.querySelectorAll(selector));
  }
}

// onclick-class.js

import { DomElement as DE } from './dom-element';

export class DomElement extends DE {
    constructor(elements) {
	super(elements);
    }
	
    onClick(callback) {
	onClick.call(this, callback);
    }
}

function onClick(callback, elements) {
    if(!elements) {
      elements = this.elements;
    }
    if(!Array.isArray(elements)) {
      elements = [elements];
    }
    elements.forEach(element => element.addEventListener('click', callback));
}

// custom.js

var h1 = new DomElement('h1');
h1.onClick(() => console.log('click'));

```

Ok, so the first thing to note is by aliasing the class name I can preserve it with each extension, which is really nice. I can also preserve my composition pattern. But I can only add one extension at a time which is why there's no add style method in the above. So how could I include both extension classes? Because currently there is no syntax that supports multiple inheritance in the ES6 class syntax.

```javascript

// onclick-class.js

export class OnClick {	
    onClick(callback) {
	    onClick.call(this, callback);
    }
}

// style-class.js

export class AddStyle {	
    addStyle(style) {
	    addStyle.call(this, style);
    }
}

// base-class.js

import { OnClick } from 'onclick-class';
import { AddStyle } from 'style-class';

export class DomElement extends OnClick, AddStyle {
  constructor(selector) {
    this.elements = Array.from(document.querySelectorAll(selector));
  }
}

```

The above comma separated extends syntax doesn't work. But maybe I can manually change that. Extends accepts an expression as well as a function / class which gives me a chance to create a workaround. As I have a go at this I'll show the full contents of my individual files again just to show exactly what is going on.

```javascript

// style-class.js

function addStyle(style, elements) {
  if(!elements) {
    elements = this.elements;
  }
  if(!Array.isArray(elements)) {
    elements = [elements];
  }
  var key = Object.keys(style)[0];
  elements.forEach(element => element.style[key] = style[key]);
}

export class AddStyle {	
  addStyle(style) {
    addStyle.call(this, style);
  }
}

// onclick-class.js

function onClick(callback, elements) {
    if(!elements) {
      elements = this.elements;
    }
    if(!Array.isArray(elements)) {
      elements = [elements];
    }
    elements.forEach(element => element.addEventListener('click', callback));
}

export class OnClick {	
    onClick(callback) {
	    onClick.call(this, callback);
    }
}

// combine-utility.js

export function combine(...constructors) {
  let combined = function() {};
  combined.prototype = constructors.reduce(function(proto, constructor) {
    Object.getOwnPropertyNames(constructor.prototype).forEach(function(key) {
      if(key !== 'constructor') {
        proto[key] = constructor.prototype[key];
      }			
    });
    return proto;
  }, {});
  return combined;
}

// base-class.js

import { OnClick } from 'onclick-class';
import { AddStyle } from 'style-class';
import { combine } from 'combine-utility';

export class DomElement extends combine(OnClick, AddStyle) {
  constructor(selector) {
    this.elements = Array.from(document.querySelectorAll(selector));
  }
}

// custom.js

var h1 = new DomElement('h1');
h1.onClick(() => console.log('click'));
h1.addStyle({ color : 'red' });

```

Woohoo! Success! I've got individual functions that would work inside or outside of a class. I've managed to create a function that combines the non-constructor prototype properties from the classes and combine them onto a single function which I extend the base class with. As long as the extension classes don't have a constructor this pattern should work just fine. In the example of DOM element modification this pattern works well but I'm sure there will be scenarios in which it doesn't. 

I should also note at this time that I've changed my Gulp step ever since I've been exporting and importing to use Webpack. Here's my build step:

```javascript

gulp.task('js:base', () => {
    
    let entry = {};
    entry['entry'] = 'js/dom-element.js';
    let config = Object.assign({}, webpackConfigSrc, { entry });
    
    return gulp.src('./src/js/dom-element.js')
        .pipe(plumber())
        .pipe(webpackStream(config))
        .pipe(rename('dom-tool.js'))
        .pipe(gulp.dest(jsDest));

});

```








