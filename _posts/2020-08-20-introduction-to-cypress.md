---
layout: post
title: Introduction to Cypress
date: 2020-08-20
tags: [ Cypress, Javascript ]
---

[Cypress](https://www.cypress.io/) is a lightweight framework made by frontend developers to test UI applications. Cypress is gaining popularity in a world where [Selenium](https://www.selenium.dev/) is the master because it's easy to use, it's fast and really user friendly. 

It supports debugging, repeat tests, headless mode and many more features just by using the Cypress browser. But it's limited to write tests in JavaScript using the [Mocka](https://mochajs.org/) test runner which should not be a limitation at all.

## Setting Up Cypress

**1.- Install [NodeJS](https://nodejs.org/en/) higher than 8.x**
**2.- Create the _package.json_ file with content:**

```json
{
  "name": "tests",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "cypress": "cypress open",
    "test": "cypress run"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

Using the above configuration, we can then run our tests using either "npm run cypress" or "npm test".

**3.- Install Cypress:**

```sh
npm install cypress
```

Cypress will create the folder structure:

- cypress.json - to configure Cypress
- cypress/fixtures - to configure static data to be used by our tests
- cypress/integration/examples - to have the test files
- cypress/plugins - to customise the Cypress behaviour (more in [here](https://docs.cypress.io/guides/tooling/plugins-guide.html#Use-Cases))
- cypress/screenshots - to see the screenshots or recordings made by Cypress when running the tests

**4.- Run cypress:**

```sh
npx cypress open
```

This will open a browser to run the example tests. 

## First Test Case

Let's write our first test in _cypress/integration/<name_of_test>.spec.js_. In Cypress, we can have either:

- action or commands: "cy.get(...).type" or ".click" or ...
- validations or assertions: "cy.get(...).should"

For this test, we're going to visit a TODO app that is part of the installed Cypress examples:

_cypress/integration/todoapp.spec.js_:
```js
/// <reference types="cypress" />

describe('todo actions', () => {

  beforeEach(() => {
    // given
    cy.visit('http://todomvc-app-for-testing.surge.sh') // browse to this URL
    cy.get('.new-todo').type("Clean room{enter}") // focus the input with class name new-todo, type some characters and click enter
  })

  it('should add a new element to the list', () => {
    // assert
    cy.get('label').should('have.text', 'Clean room') // validate the label is created with expected content
    cy.get('.toggle').should('not.to.checked') // validate that the toggle is not checked
  })

  it('should mark completed an element', () => {
    // when
    cy.get('.toggle').click() // mark the item as completed
    // assert
    cy.get('label').should('have.css', 'text-decoration-line', 'line-through') // validate the label is updated
  })

  it('should clear completed elements', () => {
    // given
    cy.get('.toggle').click() // mark the item as completed
    // when
    cy.contains('Clear completed').click() // Search an element with a text containing 'Clear completed' and click it.
    // assert
    cy.get('.todo-list').should('not.have.descendants', 'li') // Validate that the are no more items
  })
})
```

| Note this is only an example, for further commands, visit [the official Cypress documentation](https://docs.cypress.io/api/api/table-of-contents.html).

### Run our tests!

Cypress supports two ways to run the tests:

**- Interactively** - see the actions in the browser

```sh
npx cypress open
```

And select our new test file and run it.

**- Non Interactively (headless mode)** - see the test statuses via command line

```sh
npx cypress run
```

We can see a rendered video to check what happened in each test.

### Improve tests using Page Objects

The idea is that instead of doing:

```js
cy.get('.new-todo').type("Clean room{enter}")
```

We want to have something with:

```js
todoPage.addTodo('Clean room')
```

In order to achieve this, we need to create our page object class _cypress/page-objects/todo-page.js_:

```js
export class TodoPage {

  // browse to the TODO site
  navigate() {
    cy.visit('http://todomvc-app-for-testing.surge.sh')
  }

  // focus the input with class name new-todo, type some characters and click enter
  addTodo(todoText) {
    cy.get('.new-todo').type(todoText + "{enter}")
  }
}
```

And our tests would look like as:

_cypress/integration/todoapp.spec.js_:
```js
/// <reference types="cypress" />

import { TodoPage } from "../page-objects/todo-page";

describe('todo actions', () => {

  const todoPage = new TodoPage()

  beforeEach(() => {
    // given
    todoPage.navigate()
    todoPage.addTodo('Clean room')
  })

  // ...
})
```

| Note that we could export directly our functions if our page object does not share any data among tests.

## Tips

**- Use the Cypress browser to inspect the element you want to test**

```js
cy.get('<the element that I inspected in the Cypress browser>')
```

**- Delay elements in the UI**

What if my web is rendered but some of our components are loaded asynchronously? The way we can simulate this scenario is by sending a delay parameter when we browse to our web:

```js
cy.visit('http://todomvc-app-for-testing.surge.sh/?delay-<MY_CLASS_ELEMENT=3000') // 3 sec
```

Also, we can configure the timeout to wait until a component appears:

```js
cy.get('.<MY_CLASS_ELEMENT', {timeout: 6000})
```

**- Run only one test of my group**

We can mark a mocha test with **it.only**:

```js
it.only('should ... 1', () => {
  // ...
})

it('should ... 2', () => {
  // ...
})
```

In this case, only the first test will be executed. Also, we can mark other tests with only.

**- Disable for file changes**

Go to cypress.js and add this settings:

```js
{
  "watchForFileChanges": false
}
```

This way Cypress won't load our tests automatically.