# Hkxx26 Computer Graphics Coursework.

## How to Use

To Run This file you must have node.js installed. 
If you do not have it installed you can download a copy here: 

    https://nodejs.org/en/download/
    
Once Downloaded We require the node package 'http-server' .
Type the following command into command-line to install:

    npm install -g http-server
            
Now we are ready to view the living room. 

On the Command-line navigate to the folder holding the code using:

    cd '<PATH_TO_FOLDER>/hkxx26' (Replace <PATH_TO_FOLDER> with actual path)

Check all the files in the file structure section below are present. (IMPORTANT) 

Now Type:
    
    http-server
    
This will start a local http-server. In the browser of your preference (Chrome Recommended) go to the url:
    
    http://127.0.0.1:8080/

From here we can view the living room. The controls are:
    
    Left/Right Arrow Keys : Move the camera left and right.
    Up/Down Arrow Keys : Move the camera up and down.
    
    x/z Arrow Keys : Move the chairs In/Out
    c/v Arrow Keys : Rotate the lamp Clockwise/Anticlockwise

## File Structure

    The file structure of this project is as follows (Make sure all files are present):
    
    ./hkxx26
      |-index.html (Holds the HTML code)
      |-index.js (WebGL JS Code)
      |-img (Image Assets Folder)
        |- Floor.jpg
        |- Light.jpg
        |- Sofa.jpg
        |- TV.jpg
        |- Walls.jpg
        |- Wood.jpg
      |-lib (WebGL Libraries)
        |- cuon-matrix.js
        |- cuon-utils.js
        |- initShaders.js
        |- webgl-debug.js
        |- webgl-utils.js
      

## Scene Graph

    The scene graph structure of this project is as follows:
    
    ./canvas
      |-Floor & Walls (Group)
        |- Floor
        |- Wall 1
        |- Wall 2
        
      |-Table & Chairs (Group)
        |-Table
          |- Base
          |- Under-Base
          |-Legs (2x Group) (One front and back)
            |- Table Leg
            
        |-Chairs (4x chairs)
          |-Opposite (2x) (2 Each each opposite sides of the table)
            |-Side (2x) (Two chairs side by side on each side of the table)
              |- base
              |-Legs (4x Chair Legs)
                |-Opposite (2x) (One one each opposite of the Chair)
                  |-Side (2x) (Two chairs side by side)
                    |- Chair Leg
              

      |-Standing Lamp
        |- Base
        |- Column
        |- Bulb 
        
      |-TV Corner (Group)
        |-TV
          |- Drawer 
          |- Mount 
          |- TV
        |-Sofa
          |- Bottom 
          |- Back 
          |- Right Armrest 
          |- Left Armrest 
        |- Chair
          |- Bottom 
          |- Back 
          |- Right Armrest 
          |- Left Armrest 