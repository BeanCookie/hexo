---
title: Java 规则引擎Easy Rules
date: 2021-11-10 17:42:31
categories:
- Java
tags: 
- 规则引擎
---

### 什么是规则引擎
*马丁福勒* 曾给过如下定义

> 规则引擎就是提供一种替代的计算模型。与通常的命令式模型不同，命令式模型由带有条件和循环的顺序命令组成，规则引擎基于[生产规则系统](https://martinfowler.com/dslCatalog/productionRule.html)。这是一组产生式规则，每个规则都有一个条件和一个动作 - 简单地说，您可以将其视为一堆 if-then 语句。
>
> 精妙之处在于规则可以按任何顺序编写，引擎会决定何时使用对顺序有意义的任何方式来计算它们。考虑它的一个好方法是系统运行所有规则，选择条件成立的规则，然后执行相应的操作。这样做的好处是，很多问题都很自然地符合这个模型：
>
> ```
> if car.owner.hasCellPhone then premium += 100;
> if car.model.theftRating > 4 then premium += 200;
> if car.owner.livesInDodgyArea && car.model.theftRating > 2 then premium += 300;
> ```
>
> 规则引擎是一种工具，它使得这种计算模型编程变得更容易。它可能是一个完整的开发环境，或者一个可以在传统平台上工作的框架。生产规则计算模型最适合仅解决一部分计算问题，因此规则引擎可以更好地嵌入到较大的系统中。
>
> 你可以自己构建一个简单的规则引擎。你所需要做的就是创建一组带有条件和动作的对象，将它们存储在一个集合中，然后遍历它们以评估条件并执行这些动作。

### 使用Easy Rules定义规则

大多数业务规则可以用以下定义表示：

- Name : 一个命名空间下的唯一的规则名称

- Description : 规则的简要描述
- Priority : 相对于其他规则的优先级
- Facts : 事实，可立即为要处理的数据
- Conditions : 为了应用规则而必须满足的一组条件
- Actions : 当条件满足时执行的一组动作 

Easy Rules中可以使用多种方式定义规则，使用过程中可以根据情况使用最合适的方式。

1. Java注解方式

   ```java
   @Rule(name = "weather rule", description = "if it rains then take an umbrella")
   public class WeatherRule {
   
       @Condition
       public boolean itRains(@Fact("rain") boolean rain) {
           return rain;
       }
       
       @Action
       public void takeAnUmbrella() {
           System.out.println("It rains, take an umbrella!");
       }
   }
   ```

   

2. 使用链式API方式

   ```java
   Rule weatherRule = new RuleBuilder()
           .name("weather rule")
           .description("if it rains then take an umbrella")
           .when(facts -> facts.get("rain").equals(true))
           .then(facts -> System.out.println("It rains, take an umbrella!"))
           .build();
   ```

   

3. 使用Java表达式

   ```java
   Rule weatherRule = new MVELRule()
           .name("weather rule")
           .description("if it rains then take an umbrella")
           .when("rain == true")
           .then("System.out.println(\"It rains, take an umbrella!\");");
   ```

   

4. 使用EL表达式

   ```java
   public static void printRain(String message) {
        System.out.println(message);
   }
   
   Rule weatherRule = new SpELRule()
     .name("weather rule")
     .description("if it rains then take an umbrella")
     .when("#{ #rain == true }")
     .then("#{ T(example.RuleApp).printRain(\"It rains, take an umbrella!\") }");
   
   
   ```

   

5. 使用配置文件定义规则

   ```yaml
   # weather-rule.yml
   name: "weather rule"
   description: "if it rains then take an umbrella"
   condition: "rain == true"
   actions:
     - "System.out.println(\"It rains, take an umbrella!\");"
   ```
   
   
   
   ```java
   MVELRuleFactory ruleFactory = new MVELRuleFactory(new YamlRuleDefinitionReader());
   Rule weatherRule = ruleFactory.createRule(new FileReader("weather-rule.yml"));
   ```
   

### 在Easy Rules中使用规则

```java
public class Test {
    public static void main(String[] args) {
        // define facts
        Facts facts = new Facts();
        facts.put("rain", true);
        // facts.put("rain", false);

        // define rules
        Rule weatherRule = ...
        Rules rules = new Rules();
        rules.register(weatherRule);

        // fire rules on known facts
        RulesEngine rulesEngine = new DefaultRulesEngine();
        rulesEngine.fire(rules, facts);
    }
}
```

### 组合规则

CompositeRule由一组规则组成。这是一个典型地组合设计模式的实现。组合规则是一个抽象概念，因为可以以不同方式触发组合规则。

Easy Rules自带三种CompositeRule实现：

1. UnitRuleGroup : 要么应用所有规则，要么不应用任何规则（AND逻辑）
2. ActivationRuleGroup : 它触发第一个适用规则，并忽略组中的其他规则（XOR逻辑）
3. ConditionalRuleGroup : 如果具有最高优先级的规则计算结果为true，则触发其余规则

```java
Rule weatherRule1 = new RuleBuilder()
        .name("weather rule1")
        .description("if it wind then add clothes")
        .when(facts -> facts.get("rain").equals(true))
        .then(facts -> System.out.println("It wind, add clothes!"))
        .build();

Rule weatherRule2 = new SpELRule()
        .name("weather rule2")
        .description("if it rains then take an umbrella")
        .when("#{ #rain == true }")
        .then("#{ T(example.RuleApp).printRain(\"It rains, take an umbrella!\") }");

Facts facts = new Facts();
facts.put("rain", false);
facts.put("wind", false);
// facts.put("rain", false);
// facts.put("wind", false);

UnitRuleGroup ruleGroup = new UnitRuleGroup();
ruleGroup.addRule(weatherRule1);
ruleGroup.addRule(weatherRule2);

// define rules
Rules rules = new Rules();
rules.register(ruleGroup);

// fire rules on known facts
RulesEngine rulesEngine = new DefaultRulesEngine();
rulesEngine.fire(rules, facts);
```

#### Links

https://martinfowler.com/bliki/RulesEngine.html
