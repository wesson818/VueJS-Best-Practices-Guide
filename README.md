# Deverus Vue.js Style Guide
### Guide for developing Vue.js applications. 

v. 0.0.1
---

Vue.js is an amazing framework, which can be as powerful as Angular or React, the two big heavy hitters in the world of front-end frameworks.  

However, most of Vue's ease-of-use is due to the use of Observables - a pattern that triggers re-renders and other function calls with the reassignment of a variable.  

Because of this, while Vue is very powerful, it can be very easy to do some very dumb things with it - things Vue will allow you to do.  The irony of Vue's lower learning curve is that while it is easier to do most things in Vue, it's also easier to make serious mistakes that can make adding features or debugging harder.  

This is the styleguide I've put together for the team at Deverus to help organize ourselves as we move from a primarily server-side rendered, Cold-Fusion based view to dynamic SPAs using Vue. 

## Section 1: The Basics

Vue.js has put out an [official style guide](https://vuejs.org/v2/style-guide/) and we should consider this document an extension of that one.  While this shouldn't be an exhaustive list, here are some key takeaways: 

* Component data() must be a function, so that components can be reused with new instances of components. 
* Prop definitions should be very detailed, including, if possible, 'type', 'required', and 'validator' information. 

Bad:
```javascript 
Vue.component({
  props: ['status']
})
```

Good: 
```javascript
Vue.component({
  props: {
    status: {
      type: String,
      required: true,
      validator: (value) => ['syncing', 'synced', 'error'].includes(value); 
    }
  }
})
```

* Always use key with v-for.
* No more than one component per file. 
* Use PascalCase for naming component files. (i.e. "MyComponent.vue" not "myComponent.vue")
* Base components (that is, a "dumb" component that has no interactivity and just displays data) should begin with the prefix "Base". (As in : "BaseButton.vue", "BaseTable.vue")
* Components that are created and used only once should be prefixed with "The", as in "TheHeading.vue, "TheSidebar.vue"  
* Child components tightly coupled with their parent should include the parent component name as a prefix.  For example, "TodoList.vue" has children "TodoListItem.vue" & "TodoListAddButton.vue"
* Directives and template expressions should only contain the most simple of javascript commands. 

Bad: 
```html
<template>
  <div>
    <app-some-component @click="val => val.reduce((pv, cv) => {
      pv[cv.id] = cv;
      return pv; 
    }, {})">
    </app-some-component>
      {{ fullName.split(' ').map(function (word) {
        return word[0].toUpperCase() + word.slice(1)
      }).join(' ')
    }}
  </div>
<template>
```

Good: 
```html
<template>
<div>
  <app-some-component @click="normalize"></app-some-component>
  {{wordmangler}}
</div>
</template>

// component
<style>
Vue.component({
  computed: {
    wordmangler: () => this.fullName.split(' ').map(function (word) {
        return word[0].toUpperCase() + word.slice(1)
      }).join(' ')
  },
  methods: {
    normalize: (event) => event.target.value.reduce((pv, cv) => {
      pv[cv.id] = cv;
      return pv; 
    }, {})
  }
})
</style>
```

Better: 
```html
// template
<template>
  <app-some-component @click="normalize">
    {{wordmangler}}
  </app-some-component>
</template>

// javascript
<script>
const normalizeData = (event) => event.target.value.reduce((pv, cv) => {
    pv[cv.id] = cv;
    return pv; 
  }, {})

const mangleWord = (val) => val.split(' ').map(function (word) {
      return word[0].toUpperCase() + word.slice(1)
    }).join(' '),

// component

Vue.component({
  computed: {
    wordmangler: () => mangleWord(this.fullName),
  }, 
    normalize: (val) => normalizeData(val), 
  methods: {
  }
})
</script>
```

* Don't mutate props *directly* in a child component. They can still be mutated, but they should use the .sync/emit pattern. 

With .sync/emit (and required props), you can allow even your dumb components to have interactivity while still allowing parent components to dispatch actions. 

Bad
```javascript
// template
Vue.component('TodoItem', {
  props: {
    todo: {
      type: Object,
      required: true
    }
  },
  methods: {
    removeTodo () {
      var vm = this
      vm.$parent.todos = vm.$parent.todos.filter(function (todo) {
        return todo.id !== vm.todo.id
      })
    }
  },
  template: `
    <span>
      {{ todo.text }}
      <button @click="removeTodo">
        X
      </button>
    </span>
  `
})
```

Good
```javascript
Vue.component('TodoItem', {
  props: {
    todo: {
      type: Object,
      required: true
    }
  },
  template: `
    <span>
      {{ todo.text }}
      <button @click="$emit('delete')">
        X
      </button>
    </span>
  `
})
```

Example
```html
// parent template
  // syntax 1
  <app-child-component foo="foo" @update:foo="val => foo = val">
  // syntax 2
  <app-child-component :foo.sync="foo">
  // syntax 2 is syntactic sugar for syntax 1, they are identical. 

// child
<script>
  methods: {
    changeFoo: (newFoo) => {
      this.$emit('update:foo', newFoo); 
    }
  }
</script>
```

## Section 2: Advanced

Following these guidelines should help you organize your apps on a more macro scale. 

These are the rules that we've come up with - and your mileage my vary.  While the above "style guide" deals more with how to write Vue code, these are *general* tips for organizing *any* SPA application, but just written with vue in mind. 

* NEVER modify the Vue prototype.
* If you HAVE to modify the prototype, extend the Vue class and modify THAT prototype.
* NEVER store extra constant values on the Vue prototype. 

Speaking from experience here.  In some legacy software we have, configuration information is assigned to Vue.prototype.$config, and all the API calls are assigned to Vue.prototype.$API - so that they're accessible as this.$config and this.$API in each Vue instance.  DON'T DO THIS.  

For constant data information, by all means, create a configuration file with the various bits of data, but you can use ES6 syntax and import in order to grab only those bits you need in the component that you need them from.  With webpack's tree shaking and code splitting, you'll also get a performance benefit. 

### Vuex 
* Use Vuex for data that A) comes from the API, and/or B) needs to be accessed by the entire application. 

The Vuex store is great because it can provide data to every component in your application, without having to create a rat's nest of callbacks.  However, it can be difficult determing *what* should be in the store.  

As a general rule, data that interacts with the API should be stored in the Vuex store. This is partially because API data is typically the kind of data used in multiple places in an application, but keeping all the API data in a single place will help you when tracking down where the data is made. 

While the store *might* contain information about the state of your front-end application, if you keep all your API calls in actions, and those actions then commit to the mutator, and thus to the state, you can ensure that A) all components are pulling from the single source of truth, and that no data will be duplicated, B) your front-end store will stay synced with your back-end data, C) your application will update automatically as soon as the API call resolves.  

* Vuex stores have four main parts: the state object itself, getters, actions, and mutations.

#### Store:

* Whenever you can, normalize data that comes in array form *unless the order is important* (such as when you're getting the top 20 most recent pages from your API.) Even if the data has come back from the API as an array of objects, AND your view needs to display it as an array mapped over with a v-for directive, you should consider storing it in the store as an object of objects, by some identifying key, and then just denormalizing the data back again inside the getter. 

Bad:
```javascript
const initialState = {
  partnersList: [], // array of all partners
  currentPartner: {}, // current partner object.
};
```
Good:
```javascript
const initialState = {
  partnersList: {}, // object with all partners keyed by ID
  currentPartnerId: ``, // a string contining the ID of the current partner.
};
```

There are a couple of advantages to this.  First, you can easily use ES6+'s built in Object.keys, Object.values, and Object.entries methods to get an array back from any object, but it's a bit more complicated to go the other way around.  

Second, as an algorithm, retrieiving something from an object by key is O(1), compared to O(n) if you use searches or filters to find the same data.  It's not much of an overhead, but it adds up. 

Third, listing by key allows you to be able to access the correct data even as data is removed or added. 

Take for example, the information about partners.  Each partner has it's own entry in the database, this comes back to us in the form of an array.  These are stored in the vuex store as "partnersList".   You could switch between partners by searching for some criteria, then making a copy of the current partner in a currentPartner object in the store, but that not only is a slow method, it needlessly copies data.  

Instead, normalize the partner list by creating an object where each key is the partnerId.  Then, just store the currentPartnerId as a primitive string (or number) and access the keys in partnerList that way.  

Or better still... use a getter.  

#### Getters: 

A getter is a small function which takes the state as it's single parameter and returns some aspect of the state.  

* It's a bad pattern to access the state directly, as you don't want to accidentally change the (observable) vuex store in the component.  Instead, use a getter to get the specific data you need.  This also allows you more granularity in the type of data you need. 

So that, for example, instead of needing to reference this.$store.partnersList[this.$store.currentPartnerId] in your component, you can define a getter to do that for you automatically. 

Example: 

```javascript
const getters = {
  partnersList: state => state.partnersList, // gets the partner list. 
  // returns a function which returns any partner by id. 
  getPartnerById: (state) => (id) => state.partnersList[id], 
  // gets the current partner id. 
  currentPartnerId: state => state.currentPartnerId,
  // gets the current partner object
  // (even though that object isn't stored directly in the state)
  currentPartner: state => state.partnersList[state.currentPartnerId],
};
```

* Getters should not mutate data. (indeed, the purpose of a getter is to provide a "safer" way to access the vuex store values).  They should only return the data you need.  

* Getters are great for **taking the data from the store then formatting it the way you need it** before handing it off to the Vue component.  This keeps transformation logic outside of the presentational Vue component.  You can also use this pattern to create different transforms on the same data without mutating that data. 

#### Actions:

An action is a function which takes as it's first parameter an object with a "commit" function.  This commit function, in turn, takes a string as it's first parameter - that string determines what command the mutator should run.  

Actions can be synchronous *or* asynchronous. Because the commit is passed in as a callback, you can wait for other functions to resolve before dispatching the data to the store.  

Example: 

```javascript
import * as types from '../mutationTypes'

const actions = {
  setCurrentPartner: ({ commit }, partnerId) => {
    commit(types.SET_CURRENT_PARTNER_ID, partnerId);
  },
  grabPartnerList: ({ commit }) => {
    return api.get.partnersList()
      .then(partnersList => {
        commit(types.SET_PARTNERS_LIST, normalizePartners(partnersList));
      })
      .catch((err) => {
        console.error(err);
      });
  },
};
```

Actions don't actually change the store data - that's what mutators are for - but they send the command to change data so that it's ready for inclusion in the store.  

In the above example, it can be very simple - "setCurrentPartner()" takes an object with the commit method, then runs it with "SET_CURRENT_PARTNER_ID" as the first variable, and the partnerId as the second.  Most actions will be that simple. 

"grabPartnerList()" on the other hand, shows some of the complexity in the actions.  This doesn't take a second parameter, because 'grabPartnerList' is actually kicking off the API call when it's executed.  

In the case, the api.get.partnersList() returns a promise object, that has a .then method.  Only after the data has been resolved does the data get committed to the store.  

In this case, though, it's not the raw data, but a normalized version - as you can see by the function call to normalizeParameters (not pictured here) which will transform the data **after it comes from the API and before it gets to the store**

(to recap: if you need to transform data *before* it gets to the store, transform it in the action, if you need to transform data *after* it comes from the store, transform it in the getter). 

#### Mutations 

Mutations is an object with different methods that take the Observable state as the first parameter.  It is here, *and ONLY here* that any part of the state should be changed. 

Because we have put our transformation logic in the actions and getters, most mutations will be fairly simple reassignment affairs.  

Example: 
```javascript
const mutations = {
  [types.SET_PARTNERS_LIST](state, partnersList) {
    state.partnersList = partnersList;
  },
  [types.SET_CURRENT_PARTNER_ID](state, partnerId) {
    state.currentPartnerId = partnerId;
  }
};
```

As you can see, types.SET_PARTNERS_LIST and types.SET_CURRENT_PARTNER should look familiar.  That is because those are really just strings.  If we were to look in mutationTypes.js: 

mutationTypes.js
```javascript
export const SET_PARTNERS_LIST = `SET_PARTNERS_LIST`
export const SET_CURRENT_PARTNER_ID = `SET_CURRENT_PARTNER_ID`
```

What is happening is that commit uses the first parameter set to it to determine *which* of these mutator methods to execute with the values in it's other parameters.  

## In Review:

Data should be primarily in this flow...

* STEP 1: EVENT
   The View (Vue) has an *event* happen (a button push, a listener trigger, a page load)
* STEP 2: DISPATCH AN ACTION
   The View then *dispatches an action* along with any data payload (such as the component data state). 
* STEP 3: PROCESS THE DATA
   The action then processes the data from the event, and if necessary, grabs more data from the API.  All this data is put into a payload. 
* STEP 4: COMMIT THE PAYLOAD(s)
   The callback to the commit in the Action (which can happen more than once, and can be asynchronous) commits those changes to the mutator. 
* STEP 5: MUTATE THE STATE
   Based on the type put into the mutator, the state then gets mutated. 
* STEP 6: OBSERVABLES CHANGE THE VALUE OF GETTERS
   Because the value of the state has changed, the value of the getters is now recalculated based on these new state values.  
* STEP 7: GETTERS GET COMPUTED INTO THE VIEW
   The getters, imported into the component, then change the view with the updated information.  This in turn MAY trigger another event, but it is more likely that user action will trigger the next event.  


### Using Store Values in Components

As it turns out, it *is* possible to read from the store directly and to dispatch a mutation directly to the Vuex store.  *However* it is not advised.  Instead, you want to use the getters and actions to interact with the store. 

As it so happens, the creators of Vuex have created two handy components for you to use - mapActions and mapGetters.  In your component, simply: 

```javascript
import {mapGetters, mapActions} from 'vuex'

Vue.component({
  name: `PartnerList`,
  computed: {
    ...mapGetters([
      `partnersList`,
      `getPartnerById`,
      `currentPartnerId`,
      `currentPartner`
    ]),
    // ... any other computed properties... 
  }
  methods: {
    ...mapActions([
      `setCurrentPartner`,
      `getPartnersList`
    ]),
    handleChangePartner (event) => {
      let newId = event.target.value; 
      if(newId !== this.currentPartnerId){
        this.setCurrentPartner(newId); // this.currentPartner automatically updates. 
      }
    }
  }
})
```

Note the syntax.  mapGetters and mapActions take an array, then iterates over that array, and creates computed values that can be referenced by the key.  So for example, we can access the partners list with "this.partnersList" (instead of this.$store.partners.partnersList).

The action is similar - setCurrentPartner() is mapped to this.setCurrentPartner - but wait!  Didn't setCurrentPartner() take a commit object as the first parameter? That's true, but ...mapActions automatically [*curries*](https://www.sitepoint.com/currying-in-functional-javascript/) the commit object and returns a new function which only takes the remaining parameters.  

So when this.setCurrentPartner(newId) executes, not only does the currentPartnerId get changed in the store, but the value of this.currentPartner(the getter) changes instantly as well.  

## Section 3: Linting and the nitty gritty details. 

One of the reasons that Vue is considered so easy to use is the great boilerplate-maker, Vue-cli, which comes with a ready-to-go webpack configuration, babel transpilers, and eslint configuration. 

Linting is important not just for catching errors but also enforcing a universal code style that everyone at Deverus and all of our contractors can understand. 

In fact, you can actually have the code automatically make most changes for you, simply by running "yarn lint --fix" on the terminal command line.  Some (but by no means all) of the things it will pick up: 

* syntax errors
* unused variables
* undeclared variables
* misspelled variables
* lines that are over 80 characters
* convert tabs to spaces


The official Deverus guide is an extension of the airbnb eslint configuration.  The following should be used as your eslintrc.js file:

```javascript
// https://eslint.org/docs/user-guide/configuring

module.exports = {
  root: true,
  parser: "babel-eslint",
  parserOptions: {
    sourceType: "module"
  },
  env: {
    browser: true
  },
  extends: "airbnb-base",
  // required to lint *.vue files
  plugins: ["html"],
  // check if imports actually resolve
  settings: {
    "import/resolver": {
      webpack: {
        config: "build/webpack.base.conf.js"
      }
    }
  },
  // add your custom rules here
  rules: {
    // don't require .vue extension when importing
    "import/extensions": [
      "error",
      "always",
      {
        js: "never",
        vue: "never"
      }
    ],
    // allow optionalDependencies
    "import/no-extraneous-dependencies": [
      "error",
      {
        optionalDependencies: ["test/unit/index.js"]
      }
    ],
    // allow debugger during development
    "no-debugger": process.env.NODE_ENV === "production" ? 2 : 0,
    // allow paren-less arrow functions
    "arrow-parens": 0,
    // allow async-await
    "generator-star-spacing": 0,
    // strings must use double 
    quotes: ["warn", "double", { avoidEscape: true }],
    // only allow dangling commas on multiline arrays, objects
    "comma-dangle": [2, "only-multiline"],
    // allow anonymous arrow functions to use ternary returns
    "no-confusing-arrow": 0,
    // allow you to name default imports anything you want
    "import/no-named-as-default": 0,
    // allow parameter reassignment (required for vuex)
    // also useful for passing observables to functions which will
    // operate on them as side-effects.
    "no-param-reassign": 0,
    // allow console.log
    "no-console": 0,
    // maximum line length
    "max-len": [1, {
      code: 80,
      tabWidth: 2,
      ignoreComments: true,
      ignoreUrls: true,
      ignoreTemplateLiterals: true,
      ignoreRegExpLiterals: true
    }]
  }
};

```
