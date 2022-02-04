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

### Evaluating Conditional Relationship

The following script is designed to determine if a the conditions within a conditional or `IF` relationship are met. This would be used on Example 4 as it has been tuned to account for the types of relationships used within it. 

I will be explaining this script in terms of Example 4. In Example 4, there is an if relationship pointing from one node to another with multiple properties. These properties are `test_node`, `field`, `quantifier`, and `type`.  The `test_node` property is used to identify the `id` property of the node whose properties we wish to examine and from which to evaluate if our conditial relationship is satified. The `field` property dictates which property of our selected node we wish to examine to determine if the conditional is satisfied. The `quantifier` property will provide a value we will compare the test node's identified property against. The `type` property will determine which type of comparison will be made between the test node's identified property and the `quantifier`. At this time, the following script will only support the following comparisons: `<`, `>`, and `=`.

    MATCH ()-[r:IF]->() WITH r
    CALL apoc.do.case([
      r.type = "<",
      'WITH $relationship as rel MATCH (n) WHERE ID(n)=rel.test_node RETURN n.' + r.field + ' < rel.quantifier',
      r.type = ">",
      'WITH $relationship as rel MATCH (n) WHERE ID(n)=rel.test_node RETURN n.' + r.field + ' > rel.quantifier',
      r.type = "=",
      'WITH $relationship as rel MATCH (n) WHERE ID(n)=rel.test_node RETURN n.' + r.field + ' = rel.quantifier'
      ],
      'RETURN false',
      {relationship:r})
    YIELD value
    RETURN value
   
I'm going to further break down this script. 
   
The following first line of the script will locate all `IF` relationships in the graph and collect them in the variable "r". 
   
    MATCH ()-[r:IF]->() WITH r
    
The next line initiates a "do case" which means that it will perform a series of if statements and carry out a string of code for whichever statement is statisfied first. This "do case" will be carried out on each relationship collected in the variable "r". Additionally, the line following the ellipses will be the end of the "do case". This ending line included `{relationship:r}`, what this does is passes the variable r from above into the "do case" under the variable name "relationship". This will not be used in the if statements, as they have access to the variable "r" defined above without any passing of variables. "relationship" will be needed when we are assembling strings of Cypher code we would like to have execute when an if statement is satisfied. Within a string to be executed in the "do case", we do not have access to "r" without passing in "r" as the variable "relationship". (Note: the choice of the name "relationship" for the variable was arbitrary).

    CALL apoc.do.case([
    ...
    {relationship:r})
    
The next two lines work together. The first line comprises of the first if statement in our "do case". It will check if the `type` property for relationship "r" is `<`. (Due to the nature of the "do case" only one relationship is evaluated at a time but the larger variable r is used to identify what each relationship collected within it would be doing). If the `type` property of "r" is in fact `<`, the second line will be carried out. 

The second line is a concatentation of 3 parts. The first part uses `WITH $relationship as rel` to establish that we want to access the "relationship" variable we passed in earlier and refer to it by "rel" for the rest of the string. This first part then uses `MATCH (n) WHERE ID(n)=rel.test_node RETURN n.` to locate the node in the graph that has the Neo4J uniquely assigned identity number that is equal to "rel.test_node" (the `test_node` property value of the relationship we are now referring to as "rel") call it n and then return a value. This value is composed of elements of what remains of all three parts. The last section of the first part is `n.` which will end this particular first string in such a way that the concatentation with `r.field` will identify the property of node "n" we wish to examine. (Note: we can use "r" here as it is not in the string so we have access to the original r.) The next concatenation is a string that will complete the comparison `< rel.quantifier`. That is to say that `RETURN n.' + r.field + ' < rel.quantifier'` would amount practically to a boolean value as to whether the given field of "n" is < the `quantifier` property of the relationship. 

    r.type = "<",
    'WITH $relationship as rel MATCH (n) WHERE ID(n)=rel.test_node RETURN n.' + r.field + ' < rel.quantifier',
    
The next 4 lines behave similarly for the `>` and `=` cases. 

The next line will identify the end of the if statements we want to examine for our "do case". The line that follows then states that if we have exhausted the if statements and none were satisfied, we would like to return false. 

    ],
    'RETURN false',
    
Finally, we will look at the last two lines of the script. The "do case" will produce a boolean value but we will not have access to it unless we `Yield` the value produced. Finally, the yielded value will be returned.

    YIELD value
    RETURN value
   
A quick note. I have discussed previously in the repository that when you import a JSON file of nodes and relationships into Neo4J, we will want to keep the id associated with each node and relationship in order to construct the correct graph. However, it is important to note that this will be kept as a string value in a new `id` property for each node and relationship. This not the same as the `<id>` property that is assigned to a node or relationship by Neo4J. We cannot overwrite the `<id>` properties. In the above script, we use `WHERE ID(n)=` as part of the lines to be executed when if statements are satisfied. The `ID(n)` uses the `<id>` property assigned by Neo4J. If when you import the JSON file for Example 4 into Neo4J and they are not assigned 0 and 1 as their `<id>` properties, you should use the modified script below. The only change is that `WHERE ID(n)=` has been replaced in all instances by `WHERE toInteger(n.id)` which will evaluate the `id` property of the node that we assigned as an integer for the `test_node` identification. 
   
    MATCH ()-[r:IF]->() WITH r
    CALL apoc.do.case([
      r.type = "<",
      'WITH $relationship as rel MATCH (n) WHERE toInteger(n.id) = rel.test_node RETURN n.' + r.field + ' < rel.quantifier',
      r.type = ">",
      'WITH $relationship as rel MATCH (n) WHERE toInteger(n.id) = rel.test_node RETURN n.' + r.field + ' > rel.quantifier',
      r.type = "=",
      'WITH $relationship as rel MATCH (n) WHERE toInteger(n.id) = rel.test_node RETURN n.' + r.field + ' = rel.quantifier'
      ],
      'RETURN false',
      {relationship:r})
    YIELD value
    RETURN value
