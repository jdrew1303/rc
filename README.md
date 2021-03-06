# `rc` - a rule checker

`rc` is a completeness, overlap, and constraint satisfaction checker library for sets of rules. For a command-line interface to this library see [rc-cli](https://github.com/sgreben/rc-cli).


`rc` is implemented using [Z3](https://github.com/Z3Prover/z3), the automatic theorem prover built by Microsoft Research.

**Table of Contents** 
- [Installation](#installation)
    - [Maven dependency](#maven-dependency)
- [Usage](#usage)
    - [Using the builder API](#using-the-builder-api)
    - [Using the declaration API](#using-the-declaration-api)
    - [Performing analysis](#performing-analysis)

## Installation
### Maven dependency

```xml
<dependency>
    <groupId>io.github.sgreben</groupId>
    <artifactId>rule-checker</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

## Usage

`rc` is used as follows:
- Construct a set of rules as an input to rc (either via a builder API or via YAML files)
- Use `rc` to check the rules for completeness, consistency and constraint satisfaction
- Use the counterexamples generated by `rc` to improve the rules

The input to `rc` has the following structure:
- Module
    - Type definitions
    - Rule sets
        - Variable definitions
        - Rules
            - Precondition (when)
            - Postcondition (then)
        
There are two ways of constructing these objects:
- Using the builders in `io.github.sgreben.rc.Context` and `io.github.sgreben.rc.builders`to construct the objects directly.
- Building (or loading from YAML) the declaration classes in `io.github.sgreben.rc.declartions` and then compiling them.


As an example, we will express the following rules for a fictional IoT device:

- Given an enumeration type `STATE` with the values `ON`, `OFF`, `STANDBY`,
- Given the sensor inputs `temperature` (integer), `temperatureGoal` (integer),  `brightness` (real), `motion` (real),
- Given the state input variable `state` and output variable `stateOut` of type `STATE`,
- We apply the rules
    - When `temperature > 23 && (motion < 0.3 || brightness < 0.1)`
        - Then `stateOut = OFF`
    - When `temperature < temperatureGoal && motion >= 0.3`
        - Then `stateOut = ON`
    - When `temperature >= temperatureGoal && motion < 0.1`
        - Then `stateOut = state`
    - When `temperature >= temperatureGoal && motion > 0.1`
        - Then `stateOut = STANDBY`

### Using the builder API

To start, we grab an instance of the `Context` class, which we will use to access the builders:

```java
import io.github.sgreben.rc.Context;

Context context = new Context();
```

Next, we construct the enumeration type:

```java
EnumType stateType = context.buildType().enumeration()
    .withName("STATE")
    .withValue("ON")
    .withValue("OFF")
    .withValue("STANDBY")
    .build();
```

Using the type definition we can construct our variables:

```java
Variable temperature = context.buildExpression()
    .variable("temperature")
    .ofType(context.buildType().integer());
Variable temperatureGoal = context.buildExpression()
    .variable("temperatureGoal")
    .ofType(context.buildType().integer());
Variable brightness = context.buildExpression()
    .variable("brightness")
    .ofType(context.buildType().real());
Variable motion = context.buildExpression()
    .variable("motion")
    .ofType(context.buildType().real());
Variable state = context.buildExpression()
    .variable("state")
    .ofType(stateType);
Variable stateOut = context.buildExpression()
    .variable("stateOut")
    .ofType(stateType);

```

We are now ready to construct our rules:

```java
ExpressionBuilder eb = context.buildExpression();

// When `temperature > 23 && (motion < 0.3 || brightness < 0.1)`
// Then `stateOut = OFF`
Rule rule1 = context.buildRule()
    .withName("rule 1")
    .withPrecondition(eb.And(
        eb.Greater(temperature, context.buildValue().integer(23)),
        eb.Or(
            eb.Less(motion, context.buildValue().real(0.3)),
            eb.Less(brightness, context.buildValue().real(0.1))
        )
    ))
    .withPostcondition(eb.Equal(
        stateOut, context.buildValue().enumeration("OFF"))
    )
    .build();

// When `temperature < temperatureGoal && motion >= 0.3`
// Then `stateOut = ON`
Rule rule2 = context.buildRule()
    .withName("rule 2")
    .withPrecondition(eb.And(
        eb.Less(temperature, temperatureGoal),
        eb.GreaterOrEqual(motion, context.buildValue().real(0.3))
    ))
    .withPostcondition(eb.Equal(
        stateOut, context.buildValue().enumeration("ON"))
    )
    .build();

// When `temperature >= temperatureGoal && motion < 0.1`
// Then `stateOut = state`
Rule rule3 = context.buildRule()
    .withName("rule 3")
    .withPrecondition(eb.And(
        eb.GreaterOrEqual(temperature, temperatureGoal),
        eb.Less(motion, context.buildValue().real(0.1))
    ))
    .withPostcondition(eb.Equal(stateOut, state))
    .build();

// When `temperature >= temperatureGoal && motion > 0.1`
// Then `stateOut = STANDBY`
Rule rule4 = context.buildRule()
    .withName("rule 4")
    .withPrecondition(eb.And(
        eb.GreaterOrEqual(temperature, temperatureGoal),
        eb.Greater(motion, context.buildValue().real(0.1))
    ))
    .withPostcondition(eb.Equal(
        stateOut, context.buildValue().enumeration("STANDBY"))
    )
    .build();

```

To check properties of a set of rules, we construct an instance of the `RuleSet` class:

```java
RuleSet myRuleSet = context.buildRuleSet()
    .withName("my rule set")
    .withRule(rule1)
    .withRule(rule2)
    .withRule(rule3)
    .withRule(rule4)
    .build();
```

Done! You can now skip ahead to [Performing analysis](#performing-analysis) to learn how to check the rule set for completeness, overlap, and constraint satisfaction.

### Using the declaration API

Save the following as a file `myModule.yaml`.

```yaml
name: myModule
types:
    STATE:
      values:
        - ON
        - OFF
        - STANDBY
ruleSets:
    - name: my rule set
      variables:
        temperature: int 
        temperatureGoal: int 
        motion: real
        brightness: real
        state: STATE
        stateOut: STATE
      rules:
        - name: rule 1
          when: temperature > 23 && (motion < 0.3 || brightness < 0.1)
          then:
            stateOut: OFF
        - name: rule 2
          when: temperature < temperatureGoal && motion >= 0.3
          then:
              stateOut: ON
        - name: rule 3
          when: temperature >= temperatureGoal && motion < 0.1
          then:
              stateOut: state
        - name: rule 4
          when: temperature >= temperatureGoal && motion > 0.1
          then:
              stateOut: STANDBY
```

We can now load this module using the `ModuleDeclaration` class:

```java

import io.github.sgreben.rc.declarations.ModuleDeclaration;

ModuleDeclaration moduleDeclaration = ModuleDeclaration.load("myModule.yaml")
```

To perform analysis on the module, we have to *compile* it using a `Context`:

```java
Context context = new Context();
Module module = moduleDeclaration.compile(context);
```

We can now obtain the rule set we defined above:

```java
RuleSet myRuleSet = module.ruleSet("my rule set");
```

Done! You can now continue to [Performing analysis](#performing-analysis) to learn how to check the rule set for completeness, overlap, and constraint satisfaction.


### Performing analysis

We are now ready to check our rules for completeness, overlap, and constraint satisfaction:

```java
// Checks completeness, prints example unmatched values if incomplete
if (!myRuleSet.isComplete()) {
    System.out.println(myRuleSet.completenessCounterexample())
}
```

```java
// Checks for overlaps, prints example values matched by multiple rules
if (myRuleSet.isOverlapping()) {
    System.out.println(myRuleSet.overlaps())
}
```

```java
// Checks if the given constraint is satisfied in all cases
// Constraint: motion > 0.1 => state != OFF
Expression constraint = eb.Implies(
    eb.Greater(motion, context.buildValue.real(0.1)),
    eb.NotEqual(state, context.buildValue.enumeration("OFF"))
);
    
if (!myRuleSet.satisfiesConstraint(constraint)) {
    System.out.println(myRuleSet.constraintCounterExamples(constraint))
}
```
