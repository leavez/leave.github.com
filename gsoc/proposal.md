---
layout: page
title: Proposed Project
permalink: /gsoc/proposal/
---

shrutirijhwani@gmail.com

###Undo Functionality for Blocks Editor

####Abstract

This project proposes an implementation that would allow 'undo' operations in the Blocks editor. The Blocks editor allows an App Inventor user to build the control flow of the application being 'invented'. Currently, users have to rebuild or edit components in order to revert to previous states. The 'undo' operation proposed offers a one-step, user-friendly solution for reversing a fixed number of recent actions performed. The operation will sync with the Designer view in App Inventor.

####Introduction

An App Inventor user primarily interacts with two views of the application being built - the Designer view and the Blocks editor. The Designer view allows the user to select different components for the application as well as edit how the application looks. The Blocks editor provides simple building blocks that the user can combine and build the control flow of the application. Commonly used programming tools are represented in drag-and-drop blocks. The editor serves to make programming the application easily understandable, even to users with no development experience. The editor, therefore, plays an important part in the overall user experience of App Inventor and its user-friendliness determines whether the user wholly enjoys programming an App Inventor application.

This project would aim to build an 'undo' function in the Block editor. A single 'undo' operation is quite simply defined as a function that reverses the user's last action - consequently series of n 'undo' operations would reverse the user's last n actions ('undo' is not considered an action). The Blocks editor provides no such operation, currently. The implementation would have to account for for all possible user actions. It would also have to sync with the Designer view in App Inventor. The functionality could also be extended to create a 'redo' operation.

####Benefits

An 'undo' function is essential for an application that allows creating and editing any form of content or data. Highly improved user-experience is the primary benefit that arises from offering an 'undo' functionality.

The Blocks editor allows users to easily add programming concepts to an application, that might have been difficult to do otherwise. It also allows modification and deleting of any unwanted blocks. However, users cannot reverse any action performed in the editor - if blocks are deleted, they cannot be retrieved. Users looking to 'undo' actions would have to rebuild any deleted blocks and manually reverse changes.

This causes hindrance in the method of building applications - tweaking and experimentation - that App Inventor induces in a user, especially those with not much prior experience. An 'undo' option would intuitively fit into the Blocks editor. It would enhance the lucidity of application building that App Inventor offers. Users would be free to experiment with their code flow design, without worrying too much about quickly reversing any edits that prove unnecessary.

Apart from supporting quick and easy tweaking, the 'undo' operation will incredibly useful in reversing accidents. Both new and experienced App Inventor users could change blocks by accident - even hitting the wrong key while typing could prove disastrous. Further, new App Inventor users may not understand the editor immediately. Deleting or changing a block without fully understanding the consequences can easily be reversed with an 'undo' option.

An 'undo' option assures the user that it is safe to explore the interface. It also allows the user to incrementally create parts of the application. If the latest changes do not work as required, they can easily be undone without deciding which blocks to carefully delete or change.

The underlying advantage to these benefits lies in appreciating the user's time and effort. An accidental delete should not waste time already spent as well as time to be spent rebuilding. The 'undo' operation will, hopefully, increase efficiency of App Inventor users and allow them to invent and create with fewer hassles.

####Deliverables

A click-able 'undo' option in the Blocks editor, that performs the 'undo' function for a certain number of user actions. The operation should also be invoked with the keyboard shortcut ctrl+z or cmd+z. Effectively, reversing the latest user action should be a one-step process. The changes should reflect in the Designer view, wherever appropriate (example - deleting and renaming components).

Using this functionality to create a 'redo' operation would be a useful extension.

####Implementation Steps

#####1. Understand Blockly and Closure

The first step would be to dive into the Blockly source and understand its implementation. This process would, in turn, lead to discovering Closure and its intricacies.

A clear picture of different structures, processes and the user interface components of Blockly is necessary to understand how App Inventor uses it for creating various programming flows. It would also feature while defining a 'user action' - for instance, adding a block could be an action but moving a block within the editor may not be (as the editor does not consider the position of blocks while determining flow).

Code can be sourced from respective Blockly and Closure sources or from appinventor-sources -> appinventor -> lib.


#####2. Understand the Blocks editor

The App Inventor Blocks editor source would be the primary codebase to modify for this project. It would be necessary to figure out the different JavaScript functions as well as how blocks are stored. 'Events' in the blocks editor have to be understood.

The most important portion to comprehend would be the control flow of the Blocks editor - how blocks are transferred from the user clicks to storage. The 'undo' function would have to be implemented as part of this flow.

Code to be hacked: appinventor-sources -> appinventor -> blocklyeditor -> src


#####3. Designing the architecture of the proposed 'undo' function

Enabling 'undo' would involve

-Detecting when a user action has taken place

-Store the action

-Call 'reverse' of the action when 'undo' is called

In order to reverse a series of actions (multi-level undo), the series of actions must be stored. 


Design decisions to be taken


- Storage of sequences

'Undo' operations intuitively reverse latest actions first. This calls for a last-in-first-out data structure for storage. Stacks are commonly used in 'undo' implementations.


- Definition of an action

As mentioned earlier, moving around blocks may not have to be stored as this action does not change application behavior. On the other hand, addition and deletion of blocks defines parts of the control flow and would definitely have to be stored as actions.
Some actions might be well defined - changing the value of a logic block from 'true' to 'false'. However, while changing the value of a text block, an action is ambiguous - it could be one character typed, a word or the whole value itself.

In order to store actions, an action must be defined. This would require observing general user behavior on the Blocks editor, as well as what consequence each potential user action holds for the application. Only a set of valid actions should be stored on the 'undo' stack.
Apart from deciding what an action 'is', the 'undo' function would also need to know how to reverse a particular action. A potential design for this problem could involve the encapsulation of an action with a particular block as well as a mapping between each action and its reversal/inverse. Reversing would then boil down to performing the reverse action on the encapsulated block.


- Storage of actions

Each encapsulated action-block object (with any other required properties) needs to be stored on the 'undo' stack. The object of storage has to be designed in a space optimized manner, but objects should be easily accessible for reversal as well.

Potential implementations include integer-action-inverse action mapping for space optimization, with look-up tables based on user clicks.
Some actions might be defined as incremental and should be stored as one object on the stack - for instance, if 'undo' for text is at a word level, individual characters of each word (that are incrementally typed) should be unified in one 'undo' stack object.
Another (rather naive) implementation is storing the complete environment of the editor at each action, rather than incremental actions.
A combination of both ideas could lead to an implementation similar to dynamically scoped programming languages - stack actions (one property) as shallow binding and environment (or many properties) changes as deep binding.

In order to implement 'redo', objects are transferred from the 'undo' stack to a 'redo' stack whenever 'undo' is called. When the user performs an action, the redo stack is cleared.

In object-oriented terms, Command pattern is often used to describe how an 'undo' stack can be implemented. Commands encapsulate all necessary information that might be needed for a method call later. Each object on the stack could be considered as a command which encapsulates the action, its reversal as well as the target block.
The memento pattern allows the storage of complete states that can be used for 'undo' over many actions - like a rollback.


- How the 'undo' stack obtains required information

The 'undo' function needs to enter the Blocks editor control flow to extract and encapsulate information about the performed action.


- What happens when 'undo' is invoked by the user

Popping from the 'undo' stack, reversing and pushing to the 'redo' stack are necessary.

Forming the action reversal would involve several decisions. If the user is in the middle of another action - for instance, a logic value change has the drop down menu displayed, but the user has not clicked on the value yet - that action can be paused or aborted. Reversal would primarily involve connecting to the user interface of the blocks editor and performing required action.


- Number of objects in the stack

It is usually a fixed number or since the last manual save. Memory is an important performance consideration here.


Similar decisions would have to be taken for a 'redo' functionality as well, although design and implementation should be congruous to the 'undo' function.


#####4. Coding and testing

This will entail understanding where in the code each action occurs and how to efficiently implement its reversal. Understanding of the blocklyeditor source will be of primary importance.
The 'undo' stack and the function which will be invoked by the user could be implemented in a new class. Each object (command) on the stack would conform to an interface.

Challenges will arise when connecting each action to the stack. This would involve coding how the action is obtained (what series of user clicks), how it is stored, how it is reversed and the changes on the user interface after reversal.
A potential method of coding is to code each action that can be undone and its reverse as an individual module. All the necessary code can be tested for each action, as each module is built. As the functionality of the 'undo' stack is based on individual actions, module validation would lead to validation as a whole.

A test of correctness for each module would be defined by whether the reversal output is as expected by a user. Therefore, definition of what constitutes an action is of prime importance.
Storing states would require defining properties that encompass the environment.
Coding would require hacking the blocklyeditor source as well as defining new classes for the stack as well as commands. The Designer view sync would require hacking code from the appengine source (client side).


#####5. Final tests

Testing would involve creating random series of actions and 'undo' options and checking the correctness of the operation. This could be automated by defining the state of the editor at a given time and checking state variable values after each operation. User tests could also point out 'likely' situations where an 'undo' is invoked.


####Timeline

The timeline will refer to steps from the previous section.

#####27th March, 2015 to 24th May, 2015
- Steps 1 and 2

The duration before the Summer of Code begins will be utilized in growing familiar with Blockly, Closure and the App Inventor source. As the community bonding period coincides with this time, App Inventor sources can be better understood with help from the developer community and project mentors. Simple hacks on the Blockly editor and other relevant code will be undertaken.


#####25th May, 2015 to 14th June, 2015
- Step 3

Architecture design as well as building basic prototypes to validate decisions.


#####15th June, 2015 to 26th June, 2015
- Step 4

Code the stack and two action modules. The first module might take some understanding and time to implement. If interface design is well done, other modules could be coded and tested faster.

Mid-term evaluation begins.


#####27th June, 2015 to 2nd August, 2015
- Step 4

Code and test all action modules individually.


#####3rd August, 2015 to 17th August, 2015
- Step 5

Design and execute all final tests.


#####17th August, 2015 to 21st August, 2015
Documentation and code cleaning.
Final all-okay checks.

 

####Related App Inventor Work

The 'undo' operation for the Designer view would have to sync with the operation for the Blocks editor.

 

####Personal Note

I proposed this project as, after using App Inventor, I felt it would make the building experience more efficient and user-friendly. I am incredibly passionate about programming being a universal skill and App Inventor can be a huge step towards that. I'm new to open source, I hope to contribute fruitfully and I look forward to discussing projects with the App Inventor team.

 
