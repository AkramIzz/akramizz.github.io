---
layout: post
title: Tips for Solving Programming Problems
---

Problem solving is part of any programmer's life. You will have some restricting requirements, that force you to come with good solutions. It's relatively easy to evaluate a solution once there's one (maybe using test cases with restricted memory/time usage depending on the problem), but it's sometimes hard to think of a complete solution to a problem.

In this post, I hope by providing some of the tips I use in solving problems, whether it be a hackathon problem, a problem arised while working on a project or an interview problem.

## Tip 1: Understand the problem and the solution requirements

Most of the time, the problem statement is there. It's given, but that doesn't mean you understand what the problem is. Other times, you will have a vague feeling of what the problem is, you will know the problem, but we need to fully understand the problem, in order to solve it.

Understanding comes from first hand experience, and so what I do is to come up with examples, and how each's solution look. Given a problem, generate as many different examples as possible, and try to cover the edge cases. Once you know how a problem input looks like, and what are the expected outputs, you are already in the right path, this will help even in later steps, even before coming up with a solution this will give a great insight to any patterns in the input/output and the relation between them. If it doesn't, don't worry. There's still many steps you can go through to solve a problem, and this is where tip 2 comes in ...

## Tip 2: Think of a simplified version of the problem

I have noticed that in many hackathons (programming competitions), the problems are a modified version of another problem. Many problems are built from the ground of another problem. Often this means that the solution of your problem lies in the solution of the simpler problem. Other problems just contain two or three other problems, like an accumulation of problems, and once these subproblems are solved, it's probably the case that the whole problem is solved too.

When thinking of a simpler problem, try to extract the core essence of the problem. And If you can't simplify then divide. Try to think of simpler set of problems that when combined gives you the problem in question. Still can't do it? It's probable that you don't **understand** the problem. Still can't do it? Tip 3 surely will help.

## Tip 3: Make a brute force solution

This is the simplest algorithm one can use to solve a problem. And since code is a kind of formal language, writing a brute force solution is a great way to understand more of the problem.

There isn't much to say about this, but it surely helps. After writing a brute force solution, try and run it against as many different and diverse test input as you can, focusing on edge cases. This will cover your understanding of the problem, it's like filling the gaps to fully grasp the problem statement.

## Tip 4: Solve an example with pen and paper writing each step you take as detailed as possible

As said earier, code is a kind of formal langauge, this makes writing partial solutions and testing them a bit hard. That's why we need some informal language. And the pen and paper big-step-by-bigger-step solutions great. You have to have a mental image of what the solution should do, not a complete solution but how a human may solve the problem.

As humans, our way of solving a problem differ from computers. Take for example the problem of searching for the largest number in a small list of numbers ([102, 55, 170, 3, 7]). Humans would just take a look at the list and can, in somewhat undivisible step to humans, determine the largest number. But that doesn't apply to larger problems, and many of the steps we may take to solve a problem, can be translated to machine instructions, maybe indirectly, but it's doable.

This is also helpful with tip 2, solving an example by hand can show you the smaller steps you can take to solve the problem. Also don't forget to write the steps you take when solving, this will help you formalize the solution and sort your thoughts.

## Tip 6: Try recursion

The number of problems that can be solved with recursion is huge. Recursion is a very very powerful tool, yet very simple, so make use of it. Recursion is a kind of proof by induction, which works great for discrete data, the kind that computers operate on.

Here's how to easily write recursion algorithms. Start with the base case. That's where the problem is simple enough to give a direct solution, or a hardcoded one. Given a state, return the base case solution if the state is the base case, else, change the given state in a way to move the state towards the base case. Lastly and obviously, the solving function must call itself with the reduced state.

Reducing the state to the base state is what solves the problem and so it depends on the problem itself, but this step is made easy with some assumptions. First, all the previous subproblems can be assumed solved, and we just need to return the solution to the given state to solve the previous bigger subproblem. Secondly, When a the given state is reduced and a point is reached where the state must be reduced again in a similar fashion, call the solving function. You don't even need to understand how the full solution works, only how a subproblem is solved.

This means that we only need to worry about a partial of the problem. For example, in a tree data structure, any operation can be implemented recursivly, with the base case being that the current state's root (the state is a subtree of the original tree) is the solution. The reduction step is testing the current state's children and then calling the solving function with the children that have a potential of being the solution.