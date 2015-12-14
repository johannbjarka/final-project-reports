# Developer Guide
We are two students from Reykjavík University working on Project Discovery as a BSc Final Project for the fall semester of 2015.
The purpose of this document is to explain the code and outline the different classes that make up Project Discovery, and their functions. Furthermore, we seek to differentiate the previous work done by other students before we started working on the project in the fall semester of 2015.

## Server
The server code for the project is split into two parts, the Project Discovery part and the MMOS part. This is because the MMOS service is used only to communicate with the MMOS API, and can be used for different projects. The Project Discovery service is used to communicate to the MMOS service and the EVE Server / Database through the EVE Online client.

### Project Discovery Service
This service is what the Project Discovery client uses to get information from either the MMOS API or the EVE Server. In this section we list the important functions of this service and explain their usage.

    new_task(_charid, project_id, is_training_set=False):
        Calls the API with the given character ID and the project ID, returning the following example object:
        {
          "uid": "1446740137809-ca5d77be-cba2-4969-ab2a-94f678d76197",
          "gameId": 1,
          "playerCode": "PLAYERCODE50000",
          "projectId": 1,
          "taskId": 10001267,
          "assetsSecurityPass": "142bb6a1-4321-4ae8-886c-90f1cb3073b8",
          "isTrainingSet": false
        }
The character ID is injected automatically, developers need only worry about using the correct project ID. This function is very similar to the new_training_task function, but in that case the id of the task is a parameter and a specific task is requested. This is used in the tutorial phase where certain tasks in a certain order are needed.

    get_task_information(task_id, asset_id, security_pass):
        When the client needs an image associated with a certain task, use this function with the correct security pass that is provided when a new task object is requested. It returns the following example object:
        {
          "uid": "1446804094252-0cba2057-7257-42f0-b566-ac339121448b",
          "taskId": 10000001,
          "assets": {
            "url": "http://hpa.subcell.mmos.io/20150915154616/652_C12_1_green_blue_red.jpg?/...cont.../"
          }
        }

The asset_id specifies what color channel for the image to fetch (f.ex. channelGreenBlueRed), since this function only returns one URL to one image.

    post_classification(_charid, task, classification, duration, remark=None)
        This is called when a player has selected categories and clicked the submit button, it returns the following example object:
        {
          "uid": "1442523851681",
          "gameId": 1,
          "playerCode": "playercode123",
          "playergroupCode": "group123",
          "projectId": 1,
          "taskId": 10000001,
          "score": 0.55,
          "playerScore": 0.69,
          "playerProjectScore": 0.58,
          "reliability": 0.9876,
          "solution": [
            102
          ]
        }
        
This function was already implemented when we started, but we merged it with all rewarding functions to avoid an exploitation issue. So this function also rewards player based on the score the API returns. The rewards are a mix of ISK, LP and XP that goes towards the player's Project Discovery rank.

    player_statistics(_charid):
        This function fetches the statistics the MMOS API has collected on the player, along with their entire history of submissions. An example object looks like this:
        {
          "uid": "1446733276989-2f79fb9a-5172-4b6b-95b7-413c995cb8f9",
          "gameId": 1,
          "code": "PLAYERCODE50000",
          "createdAt": "2015-11-05T14:20:41.000Z",
          "projects": [
            {
              "id": 1,
              "classificationCount": 6,
              "classificationCountUnscored": 3,
              "score": 0.430234,
              "reliability": 0.5101164985478926,
              "history": [
                {
                  "taskId": 10000001,
                  "classificationScore": 0,
                  "playerScore": 0.430234,
                  "playerScoreChange": -0.00875409,
                  "scoredAt": "2015-11-05T14:20:51.000Z"
                },
                {
                  "taskId": 10000001,
                  "classificationScore": 0,
                  "playerScore": 0.438988,
                  "playerScoreChange": -0.00934207,
                  "scoredAt": "2015-11-05T14:20:49.000Z"
                },
                {
                  "taskId": 10000001,
                  "classificationScore": 0,
                  "playerScore": 0.44833,
                  "playerScoreChange": -0.0101388,
                  "scoredAt": "2015-11-05T14:20:47.000Z"
                }
              ]
            }
          ]
        }
        return self.MMOS.player_statistics(_charid)

This function is called every time a player submits a classification, to update their values in Project Discovery.

    get_player_state(_charid):
        This function queries the EVE database and returns the row object that matches the given character ID. The fields in that row are as follows:
        <DBRow Object [experience INT, rank INT, finishedTutorial BIT]>
   
We cache this information and update it only when a player submits a classification.

    give_tutorial_rewards(self, _charid, XP=1000):
        This function updates the experience values of a player if they have not finished the tutorial before.
        Returns a boolean determining whether they received the reward or not.
        
If the player has not received the reward before, this function calls a procedure in the database to set the relevant bit, so that the player can never get that reward again.
        
### MMOS Service
The MMOS Service is used to communicate with the MMOS API, all the functions there are either GET or POST requests using the Python requests library. All the requests are routed through a CCP Proxy and use HMAC-SHA256 authentication. The following are the main functions of the service that we feel need explanation.

    task(self, player_code, project_id, is_training_set=False):
        """
        Get a task.

        :param player_code: The id to identify the player uniquely in the game
        :type player_code: str

        :param project_id: An array of the id(s) given by MMOS for the research project(s) for identification
        :type project_id: int|list[int]

        :param is_training_set: Explicitly ask for a task from or not-from the training set
        :type is_training_set: bool

        :return: The task:
            {
                "gameId": 1,
                "playerCode": "playercode123",
                "projectId": 1,
                "taskId": 10000926,
                "assets": {
                  "url": "http://www.proteinatlas.org/images/14909/if_selected.jpg"
                },
                "isTrainingSet": true
            }
        :rtype: dict
        """
This function is used to receive a new task for the player, it returns a task that the API allocated automatically.

    get_task_information(self, task_id, asset_id, security_pass):
        """
        Gets information regarding a task, like URL's to the task images.

        All parameters are required

        :param task_id: A number that identifies the task, given by MMOS
        :type task_id: int

        :param asset_id: The color channel of the image to fetch
        :type asset_id: string

        :param security_pass: A security pass used to get access to task related assets
        :type security_pass: int

        :return: Created task information response:
            {
              "uid": "1446804094252-0cba2057-7257-42f0-b566-ac339121448b",
              "taskId": 10000001,
              "assets": {
                "url": "http://hpa.subcell.mmos.io/20150915154616/652_C12_1_green_blue_red.jpg/...cont.../"
              }
            }
        :rtype: dict
        """
    
This function is used to get one image URL for the client.

    post_classification(self, player_code, playergroup_code, task_id, result, duration, remark=None):
        """
        Create a classification (the result of the analysis)

        :param player_code: The id to identify the player uniquely in the game
        :type player_code: str

        :param playergroup_code: The id to identify the player's group uniquely in the game
        :type playergroup_code: str

        :param task_id: The id given by MMOS for the task the classification is for
        :type task_id: int

        :param result: The result specified by the task type / project
        :type result: list[int]

        :param duration: The duration in miliseconds that the player took to solve the task
        :type duration: int

        :param remark: Any remark on the classification - the info for serendipitious discovery. If only a string is
                       provided it will be transformed into a json object {remark: stringValue} before validation
        :type remark: str

        :return: Created classification response:
            {
                "gameId": 1,
                "playerCode": "playercode123",
                "projectId": 9,
                "taskId": 90000001,
                "score": 0.1234,
                "reliability": 0.9876,
                "solution": [102]
            }
        :rtype: dict
        """
This function is used to submit a classification and returns the overall player score

    player_statistics(self, player_code):
        """
        Get statistics for a player including classification count and score.

        :param player_code: The id to identify the player uniquely in the game
        :type player_code: str

        :param project_id: If given, the response will give back the last 25 score changes in the given project
        :type project_id: int

        :return: Player statistics:
            {
                "gameId": 1,
                "playerCode": "playercode123",
                "classificationCount": 10,
                "score": 3,
                "createdAt": "2012-04-23T18:25:43.511Z"
            }
        :rtype: dict
        """

## Client
This section lists the different classes that make up the Project Discovery client. Instead of listing and explaining each function here, we rather explain the function of the class and whether or not the class was already implemented when we started the project, and what modifications we made to each class.

### Window
This class defines the window that Project Discovery resides in the EVE Online world. This class was already implemented when we started, but heavy modifications was needed. Those modifications include a new window header that displays a button to access a submission history for the player, which fades down a scrollable list of items which show the date of submission for every task and how that task affected the players score.
    
### Subcellular Atlas
This class represents the main screen of Project Discovery, and utilizes the category selector class and the task image class. This shows the player the image needing classification and the category selector the player uses to make his selection. This class then sends that information along to the Project Discovery service and gets back the submission results, which the class sends down to the result and reward classes.

### Category Selector
This class defines the hexagonal category selection grid that players use to select categories. This class was already finished when we started but we added some modifications to it. The modifications include: Fixing the category exclusions as they did not exclude the correct categories, adding available and unavailable states which determine if the category is grayed out and unselectable or not. As well as adding a hexagonal texture on top of each category that is only shown in an unknown results screen, where the color linearly interpolates between red and green based on the community consensus percentage for that category.

### Task Image
This class represents the image that needs classification, this displays an interface around the image that allows players to magnify the image by hovering over it, and switching color channels back and forth, caching the images so no re-downloads are necessary. This class also plays any sounds that are related to the image.

### Training Phase
This class was entirely made by us, and is used to take players through training, where players are taken through seven levels of increasing difficulty and rewarded with in-game experience at the completion of the seventh task.
In order to lighten the difficulty and better explain the purpose of the game and the method of classification, we restrict the 27 categories down to the ones that the solution includes plus 2 random categories within each super category that the solution is in.

### Processing View
This class was already implemented and acts as the buffer between the category selection screen and the result screen. This displays an animation that counts up from 0 to 100 and then animates two large banners to expand out and reveal the result screen. The only thing we added to this class was playing an audio track when the animation plays.

### Result Window
This class was entirely made by us, and is used to display to the user a screen that shows the player the results of their category selection. There are two types of tasks, and two types of result screens. One type of task is a known task, where experts have already given their solution and Project Discovery uses these tasks to measure their accuracy. In this case the result screen shows the category selection screen and uses different textures to show the player selection vs the correct selection. The other type of task is an unknown task, where the task has no solution yet. In this case the result screen shows the category selection screen and uses different textures to show the player selection vs a heat-map of the community selections. This allows players to see if they agree with the community on certain classifications. 

### Rewards View
This class was already implemented when we entered the project, but due to a complete redesign we had to change the UI elements of this class completely. This screen shows the rewards a player gets after each submission. This also animates an experience bar to show how close a player is to leveling up.

### Dialog
This class was entirely made by us and is used to display a dialog showing professor Lundberg along with a custom message.
The dialog disables and fades out everything below it and has a button to close it. We use this to display error messages and necessary information for Project Discovery.

### Tutorial
This class was already made when we started the project, this displays a set of tooltips in succession, where a player reads the tooltip and then presses a continue button on the tooltip to show the next tooltip. The modifications we made were shortening the tooltips and splitting them up into more parts with less text. The tooltips faded out the rest of the elements in the UI except the element the tooltip was on, but the interface was still active, so the player could still play the game while in tooltip mode. We wanted to change that so we disabled every element on the screen except the one the tooltip is on. This allows players to test the feature the tooltip is describing, to get to know the controls, without it actually counting towards their score.

## Database - Project Discovery Schema
The Project Discovery client utilizes the EVE Online database to store information of players, such as their experience and rank within the game. Much of this SQL code was already implemented, but some modification was necessary for this project.

### The Players table
This table contains information about every player of Project Discovery, its fields are CharID, experience and rank. Our modification included storing a bit called finishedTutorial to be able to tell with absolute certainty that a player has finished the tutorial, in order for the large completion reward only being given out once to each player.

### The GetPlayer procedure
This procedure was completely created by us and it takes a character ID and fetches the relevant row, returning all fields.

### The SetTutorialFinished procedure
This procedure was completely created by us, it takes a character ID and update the relevant row, setting the finishedTutorial bit for that player, making sure they only receive the tutorial reward once.

### The GiveExperience procedure
This procedure was already created but needed modification because of exploitation possibilities. This procedure takes in a character ID and an experience parameter and updates the relevant player's experience, also increasing their rank if the new experience value exceeds the required experience for that rank. Our modifications included inserting a new player if the ROWCOUNT of the updating procedure is equal to 0. Previously the insert procedure was called by the Project Discovery client, and that could possibly be exploited by players, only inserting players when there is no player found when giving experience isolates the inserting feature. 
