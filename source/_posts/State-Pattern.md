---
title: State Pattern
date: 2019-08-26 21:15:29
tags:
categories: Design Pattern
---

### Introduction

An object's behaviour depends on its internal state.

1. Identify an Wrapper class (a class wraps the state). It will serve as the "state machine" from the client's perspective.
2. Create a state base class that replicates the methods of the wrapper class. Each method takes an extra parameter: an instance of the wrapper class.
3. Implement state base class
4. The wrapper class maintain a current state object
5. All client requests to the wrapper class are delegated to the current state object, and the wrapper object's this pointer is passed
6. The state method change the "current" state in the wrapper object as appropriate.
<!-- more -->
### Real world examples


[JDiameter - Diameter State Machine](https://github.com/npathai/jdiameter/blob/master/core/jdiameter/api/src/main/java/org/jdiameter/api/app/State.java)


### Example

Let's design a vending machine

identify what states are and how they are transfered to each other and draw a graph. The node is the state. The arrow is the method.

```java
public interface State {

	public void insertMoney();

	public void backMoney();

	public void turnCrank();

	public void dispense();
}
```

```java

public class NoMoneyState implements State
{
 
	private VendingMachine machine;
 
	public NoMoneyState(VendingMachine machine)
	{
		this.machine = machine;
		
	}
	
	@Override
	public void insertMoney()
	{
		System.out.println("投币成功");
		machine.setState(machine.getHasMoneyState());
	}
 
	@Override
	public void backMoney()
	{
		System.out.println("您未投币，想退钱？...");
	}
 
	@Override
	public void turnCrank()
	{
		System.out.println("您未投币，想拿东西么？...");
	}

	@Override
	public void dispense()
	{
		throw new IllegalStateException("非法状态！");
	}
}
```

```java
public class SoldOutState implements State
{
 
	private VendingMachine machine;
 
	public SoldOutState(VendingMachine machine)
	{
		this.machine = machine;
	}
 
	@Override
	public void insertMoney()
	{
		System.out.println("投币失败，商品已售罄");
	}
 
	@Override
	public void backMoney()
	{
		System.out.println("您未投币，想退钱么？...");
	}
 
	@Override
	public void turnCrank()
	{
		System.out.println("商品售罄，转动手柄也木有用");
	}
 
	@Override
	public void dispense()
	{
		throw new IllegalStateException("非法状态！");
	}
 
}
```

The vending machine (wrapper class) has the same methods as the state interface:

```java
public class VendingMachine
{
	private State noMoneyState;
	private State hasMoneyState;
	private State soldState;
	private State soldOutState;
	private State winnerState ; 
 
	private int count = 0;
	private State currentState = noMoneyState;
 
	public VendingMachine(int count)
	{
		noMoneyState = new NoMoneyState(this);
		hasMoneyState = new HasMoneyState(this);
		soldState = new SoldState(this);
		soldOutState = new SoldOutState(this);
		winnerState = new WinnerState(this);
 
		if (count > 0)
		{
			this.count = count;
			currentState = noMoneyState;
		}
	}
 
	public void insertMoney()
	{
		currentState.insertMoney();
	}
 
	public void backMoney()
	{
		currentState.backMoney();
	}
 
	public void turnCrank()
	{
		currentState.turnCrank();
		if (currentState == soldState || currentState == winnerState)
			currentState.dispense();
	}
 
	public void dispense()
	{
		System.out.println("发出一件商品...");
		if (count != 0)
		{
			count -= 1;
		}
	}
 
	public void setState(State state)
	{
		this.currentState = state;
	}
 
	//getter setter omitted ...
}
```