# Project

Description: A web application inspired by the ship log in Outer Wilds in which users can add concepts and relationships between them. Each concepts has a title and a list of bullet points (written in Markdown). These concepts are put on a canvas. This tool must be visual

Framework: VueJS

# Requirements

## R1

Status: Completed
Task: Basic canvas
Description: A canvas must be created, in which the size is initially 1024x1024, but it will eventually be larger. The canvas will contain the concepts and their relations. The user must be able to zoom in and out and move in the canvas.

## R2

Status: Completed
Task: Add concepts
Description: The user must be able to add a new concept by doing a double click on an empty part of the canvas. There must be the possibility to drag the concepts into the canvas. The initial title of the concept is "Concept", and the user can edit it by a double clicking.

## R3

Status: Completed
Task: Relate concepts
Description: There must be the possibility to link a concept to another.

## R4

Status: Completed
Task: Delete concepts
Description: By using the canc button on a concept (when it is not in editing mode) or by using the right click of the mouse, a menù must be shown in which the user can delete the concept.

## R5

Status: Completed
Task: Import/Export JSON
Description: After R2, R3 and R4, the user have the function to export the concepts and their relations. The user must also have the function to reopen the JSON files.

## R6

Status: To Do
Task: Undo/Redo
Description: When doing every management operation (concepts or relations), there must be the possibility to undo and redo the edits
