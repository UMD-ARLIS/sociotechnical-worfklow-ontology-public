# Example Scripts 

## Simple Scripts
The following script can be used on any of the examples to return all action nodes in the graph. In order to return all of a different type of node, replace `Action` with the label name. 

    MATCH (n:Action) RETURN n

Example 2 introduces the concept of a `ConditionalAction` label. The following script will return all action nodes except for those which are labeled `ConditionalAction`. In a similar vein to the previous script, `ConditionalAction` can be replaced with any label to be excluded. 

    Match (n:Action) WHERE NOT n:ConditionalAction RETURN n

Example 3 uses the label `SubAction`. Each `SubAction` has a `larger_goal` property identifying what `Action` the `SubAction` is working towards. This script will return the value of the `larger_goal` property for each `SubAction` node.

    Match (n:SubAction) RETURN n.larger_goal

## Complex Scripts

### Returning Actions that Follow A Specified Action

The following script is designed to return the nodes of `Actions` including `SubActions`, across encompassing `Activity` nodes following the node with the "Melt" description. This would be used on Example 3 as it has been tuned to account for the types of relationships used within it. 

    Match (start:Action {description: "Melt"})<-[:AFTER*]-(immediate:Action) 
    With immediate 
    Match (start:Action {description: "Melt"})<-[:START*]-(activity:Activity) 
    With activity,immediate 
    Match (activity)<-[:AFTER*]-(nextActivity:Activity) 
    With nextActivity,immediate 
    MATCH (nextActivity)-[:START]->(removedStart:Action) 
    With removedStart,immediate 
    MATCH (removedStart)<-[:AFTER*]-(removed:Action) 
    With removedStart+collect(immediate)+collect(removed) as actions 
    UNWIND actions as a
    MATCH (a)-[:START]->(subStart:SubAction) 
    with actions,subStart 
    RETURN actions, subStart

I'm going to further break down this script. 

The following line will first locate the `Action` node with the `description property` value "Melt" and assign it the variable name "start". The same line will recursively deep search for the `Action` nodes connected to "start" by the `AFTER` relationship, and collects all those found under the variable "immediate". The format "(start)<-[:AFTER]-()" identifies that we are looking for nodes that have an `AFTER` relationship which points to the identified start node. 

    Match (start:Action {description: "Melt"})<-[:AFTER*]-(immediate:Action) 

The following line, and all other lines in the format "With \<variable name\>" are used to pass variables from one Cypher statement to the next. You will see that sometimes the values passed are not immediately used in the statement to follow, but they are used in later statements and therefore must be propogated. 

    With immediate 

The following line will first locate the `Action` node with the `description property` value "Melt" and assign it the variable name "start". The same line will recursively deep search for the `Activity` nodes connected to "start" by the `START` relationship, and collects all those found under the variable "activity". There will likely be only one.  The format "(start)<-[:START]-()" identifies that we are looking for nodes that have a `START` relationship which points to the identified start node. 

    Match (start:Action {description: "Melt"})<-[:START*]-(activity:Activity) 

Again we propogate variables. 

The following line then recursively deep search for the `Activity` nodes connected to "activity" by the `AFTER` relationship, and collects all those found under the variable "nextActivity".  The format "(activity)<-[:AFTER]-()" identifies that we are looking for nodes that have an `AFTER` relationship which points to the identified activity node. 

    Match (activity)<-[:AFTER*]-(nextActivity:Activity) 

Again we propogate variables. 

The following line will then dive into each of the collected `Activity` nodes in the variable "nextActivity" to find the actions underneath of them by identifying their first `Action` node. The full set of all these first nodes are collected in the variable "removedStart". The format "(nextActivity)-[:START]->()" identifies that we are looking for nodes that have a `START` relationship pointing to them from our identified "nextActivity" nodes. 

    MATCH (nextActivity)-[:START]->(removedStart:Action)

Again we propogate variables. 

The following line will recursively deep search for the `Action` nodes connected to the "removedStart" nodes by the `AFTER` relationship, and collects all those found under the variable "removed". The format "(removedStart)<-[:AFTER]-()" identifies that we are looking for nodes that have an `AFTER` relationship which points to the identified removedStart nodes.

    MATCH (removedStart)<-[:AFTER*]-(removed:Action) 

The following use of WITH is different from the others. This line will combine the nodes under the "removedStart", "immediate", and "removed" varibles under the new varible name "actions". 

    With removedStart+collect(immediate)+collect(removed) as actions

The following lines are used in conjunction. UNWIND takes the collection of nodes in "actions" and individually runs the following Match statement on each and gives each the temporary variable name "a". The Match statement  will recursively deep search for the `SubAction` nodes connected to the "a" node by the `START` relationship, and collects all those found under the variable "subStart". The format "(a)-[:START]->()" identifies that we are looking for nodes that have a `START` relationship pointing to them from our identified "a" node. 

    UNWIND actions as a
    MATCH (a)-[:START]->(subStart:SubAction) 

Finally "actions" and "subStart" are both propogated and returned. 
