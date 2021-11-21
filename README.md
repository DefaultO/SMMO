![smmo](https://user-images.githubusercontent.com/42414542/142766190-a6dc162f-4dd3-41f4-98e8-44689b2ec54f.png)
# SMMO (Captcha Solver)

Game: https://web.simple-mmo.com/

A friend told me about this game and how there are no public up-to-date bots available for it. I took it literal and coded a bot within hours. That was like 2 weeks ago. Kept this project private until now. Well, I won't provide you with the full functional bot, as I want you to use your braincells too.

# Usage Example
You will have to figure your own way of downloading images from https://web.simple-mmo.com/i-am-not-a-bot/generate_image?uid=x, where x is the id of the slot. Instead of downloading the file, which could technically be tracked, and banned for, you could use an internal browser like I did and "screenshot" the images you see in the captcha and work with those (I used an internal browser, but the second part, I did not do - it's really up to you).

Same with getting the captcha word you are looking for. I won't include "bot logic" so some of you lazy asses use your brain instead.

You probably can input more than just one image to the ImageSource, I never bothered, but it would speed up the process definetly. Instead of up to 4 loops, you would only do one.

Candy Cane is the only asset that is a gif, so you don't have to image classify it. If what you are looking for is the candy cane, and you downloaded the file from the i-am-not-a-bot link using it's suggested file extension, and it is a gif, you know it is the candy cane and can skip everything else.
```csharp
// Too lazy to clean my imports for you, you will need some of them to use my code below.
using CefSharp;
using System;
using System.Drawing;
using System.Drawing.Imaging;
using System.IO;
using System.Net;
using System.Text;
using System.Threading;
using System.Windows.Forms;
using SMMOBML.Model;
using Microsoft.ML.Data;
using Microsoft.ML;
using System.Linq;
using System.Text.RegularExpressions;

...

// Predict what the images are - using our mashine learning dataset
int predictedIndex = -1;
float bestAcc = 0.0f;
for (int i = 0; i < image.Length; i++)
{
    if (File.Exists(Path.Combine(Properties.Settings.Default.captcha_folder, temp_captcha_folder_name, $"{i}.png")))
    {
        // Create single instance of sample data from first line of dataset for model input
        ModelInput sampleData = new ModelInput()
        {
            ImageSource = Path.Combine(Properties.Settings.Default.captcha_folder, temp_captcha_folder_name, $"{i}.png"),
        };

        // Make a single prediction on the sample data and print results
        var predictionResult = ConsumeModel.Predict(sampleData);
        var mlContext = new MLContext();
        var transformer = mlContext.Model.Load(Path.Combine(Path.GetDirectoryName(System.Reflection.Assembly.GetExecutingAssembly().Location), "MLModel.zip"), out _);
        var predictionEngine = mlContext.Model.CreatePredictionEngine<ModelInput, ModelOutput>(transformer);
        var labelBuffer = new VBuffer<ReadOnlyMemory<char>>();
        predictionEngine.OutputSchema["Score"].Annotations.GetValue("SlotNames", ref labelBuffer);
        var labels = labelBuffer.DenseValues().Select(l => l.ToString()).ToArray();
        var index = Array.IndexOf(labels, predictionResult.Prediction);
        var score = predictionResult.Score[index];
        var prediction = predictionResult.Prediction;

        // Try to clean/prevent memory leaking
        sampleData = null;
        predictionResult = null;
        mlContext = null;
        transformer = null;
        predictionEngine.Dispose();
        predictionEngine = null;
        labelBuffer = new VBuffer<ReadOnlyMemory<char>>();
        labels = null;

        Console.WriteLine($"Predicted {prediction} by {score * 100}%");
        Console.WriteLine($"[>] {prediction} == {captcha}");
        Console.WriteLine($"{score} > {minAcc.Value / 100}?");
        
        ...
        
        // Try to clean/prevent memory leaking
        prediction = null;
        
        ...
    }
}
        
...
```
Issues you will encounter using this project:
- ### Memory Leak
  - That Image Classification causes a Memory Leak when used. I tried to prevent it for my Bot and halfway succeeded in it (it only increases the RAM usage by 1MB/captcha instead of the 100MB/captcha). Since it is a preview thing, I think it's on their end to fully fix it. In Short: Their stuff doesn't get garbage collected. Set everything you can to null to add it to the garbage collection (as you can see in the example above). I only experienced it in the past from working with GFX stuff.

<br/>

Issues you will encounter on this game:
- ### Possible Bans
  - Figure out yourself how they knew that you were botting. Play less, make it more human-like.
- ### Captcha
  - That's where I am nice and provide you with one way to solve them in this repository.<br/>Images for the learnset can be found here: https://github.com/DefaultO/SMMO/blob/main/assets/captcha.zip<br/>The actual Model can be found here: https://github.com/DefaultO/SMMO/blob/main/src/SMMOBML.Model/MLModel.zip<br/>**You don't unpack/extract the Model. The project consumes it as a .zip!**

# My Captcha Approach
Their captcha consists of 4 deformed game assets that get generated by their server. And the second I saw these captchas. I knew that I wouldn't have to pay cheap working indians sitting 8 hours a day in front of their desktop, 10 cents an hour, to solve the captchas I send them.

The Problem with this sort of captcha they use in SMMO, is, that using a image classification machine learning KI, you can pretty much programatically tell what the images in the captcha are - once you have done the work to predefine what everything is and made the KI learn your learnset. Guess what, that's exactly what I did.

Microsoft's IDE *Visual Studio*, has an individual component in the installer called **ML.NET Model Builder (Preview)**. That's what I used to create the Model and generate the projects for me. Their project examples on GitHub are either outdated or don't come with the stuff you really need, throwing errors at you. Googling for it, there is an actual release of it, found here: https://dotnet.microsoft.com/apps/machinelearning-ai/ml-dotnet/model-builder

Using 4000 images (in this case), your KI can tell to 99%, what what is.
```
===============================================Experiment Results=================================================
------------------------------------------------------------------------------------------------------------------
|                                                     Summary                                                    |
------------------------------------------------------------------------------------------------------------------
|ML Task: image-classification                                                                                   |
|Dataset: C:\Users\breit\AppData\Local\Temp\7f3fe80a-5fe5-4955-9e6c-e9e7ee05409e.tsv                             |
|Label : Label                                                                                                   |
|Total experiment time : 3758,4985021 Secs                                                                       |
|Total number of models explored: 1                                                                              |
------------------------------------------------------------------------------------------------------------------

|                                              Top 1 models explored                                             |
------------------------------------------------------------------------------------------------------------------
|     Trainer                              MicroAccuracy  MacroAccuracy  Duration #Iteration                     |
|1    ImageClassification                         0,9961         0,9969    3758,5          1                     |
------------------------------------------------------------------------------------------------------------------
```
