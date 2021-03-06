---
layout: post
title: Tech Debt Tuesday - Game Strings Redux - Part 2
tags: tech
comments: on
---


Welcome back to Tech Debt Tuesday. Last week I shared with you the major pitfalls with how the Fealty code handled game strings, and some pain points you migh encounter if you are doing something similar. The two biggest problems I was facing was that (1) bad data could crash the game and (2) there was a lot of code duplication. Today I want to share with you how I modified the code to address those problems and make editing strings possible from within the game.
<!--more-->

## The Pledge

Looking at the second problem first, I ended Part 1 with showing you all the different methods that each action in the game had to implement to fetch the relevant string:

```c#
public abstract string GetNameString();
public abstract string GetIntroString();
public abstract string GetStartString();
public abstract string GetConfirmString();
public abstract string GetRunningString();
public abstract string GetGoalString();
public abstract string GetCompletedString();
public abstract string GetHistoryTitle();
public abstract string GetHistoryDetail();
```

Each action had to implement these methods in order to provide the right variables for the localized string. Each method looked something like:

```c#
public override string GetConfirmString(KGInstance kgInstance)
{
    return FPGLoc.FormatString($"{root}Confirm", (Contract is LevyContract ? "a Levy" : "a company"), TroopGoal, (Contract is LevyContract ? "serfs" : "mercenaries"));
}
```

At the moment the game has 12 different actions which meant 108 different places where I was doing data lookups to generate strings. 

> **Painful**

The first issue, the game crashing because of the fragility of using .NET's string.Format (or equivalent - $"{variable}"), was tied to a few things. First,  the number of variables in a string was fixed at compile time (designers couldnt change it), and if an error was made when entering the string then the code would explode. Secondly, the syntax for using string interpolation made it difficult to understand strings when looking at them in a file without the game context to make them apparent. For example,

```c#
<LocString key="DisbandCompanyAction.Running">Lord, my master {0} is disbanding the {1}. I expect their return in {2}.</LocString>
```

I wrote that myself and I still have a hard time knowing what {0}, {1} and {2} correspond to without looking at the code.

## The Turn

I wanted to burn down all this code with fire, and have each action only have one place where it created the data context that it needed, and provide greater flexibility for making changes to strings while the game was running. To move away from string.Format I wanted to use string.Replace() to generate a new string from a list of text tokens. That list of tokens would be generated by each action (or whatever game context was consuming the string).

The first step was to make those nine methods above not abstract so that I could remove the implementations from the concrete inheritors. The implementation in the base abstract class now look like this:

```c#
public string GetConfirmString(KGInstance kgInstance)
{
    return FPGLoc.GetLocalizedString($"{GetType().Name}.Confirm", GetContext(kgInstance).GetTokens());
}
```

The first parameter is the key to retrieve the string - which is now composed programmatically from the class name, and the object of the calling class. The GetContext(...) method is abstract and needs to be implemented by the children. That looks like this:

```c#
public override ILocContext GetContext(KGInstance kgInstance)
{
    var context = new ActionLocContext();
    context.Add("TIME_UNTIL_COMPLETE", $"{KGHelper.GetTimeSpanFromSteps(StepsRequired - StepsDone)}");

    var settlement = kgInstance.Settlements.FirstOrDefault(x => x.ID == SettlementID);
    var company = kgInstance.Companies.FirstOrDefault(x => x.ID == CompanyID);
    var character = kgInstance.Characters.FirstOrDefault(x => x.ID == CharacterID);

    context.Add("SETTLEMENT_NAME",settlement?.Name);
    context.Add("CHARACTER_NAME", character?.Name);
    context.Add("COMPANY_NAME", company?.Name);

    context.Add("TROOP_TYPE", $"{(Contract != null && Contract is LevyContract ? "serfs" : "freemen")}");
    context.Add("COMPANY_TYPE", $"{(Contract != null && Contract  is LevyContract ? "levy" : "company")}");
    context.Add("TROOP_GOAL", $"{TroopGoal}");
    context.Add("TROOPS_AVAILABLE", $"{(Contract != null && Contract  is LevyContract ? settlement?.GetResource(SettlementResourceTypes.Serfs) : settlement?.GetResource(SettlementResourceTypes.Mercenaries))}");
    context.Add("REPORT", Report);

    return context;
}
```

The LocContext contains some state about the string key and raw value (you'll see why shortly) but it is primarily a dictionary of tokens that are available to be used in the string. The backing string now can look like this:

```c#
<LocString key="DisbandCompanyAction.Running">Lord, my master {CAPTAIN_NAME} is disbanding the {COMPANY_NAME}. I expect their return in {TIME_UNTIL_COMPLETE}.</LocString>
```

The localization system consumes these tokens to do a string.Replace on the raw string.

```c#
public static string ApplyTokens(string rawString, Dictionary<string, string> tokens)
{
    var loccedString = rawString;
    foreach (var token in tokens.Where(x => !string.IsNullOrEmpty(x.Value)))
    {
        loccedString = loccedString.Replace(token.Key, token.Value);
    }

    return loccedString;
}
```

If you look at this you might be thinking, "C'mon, FealtyDev, you are still not solving the problem of requiring code changes to add new variables to the game string!". This is true, however, we have still (1) liberated our strings from the tyranny of string.Format, (2) made the tokens readable out of context, and (3) removed the code duplication from our actions. The context is generate in one place for all of the string usage. (This means that some tokens are empty in some cases, and in that case we can safely ignore them)

Hopefully you can see the new system as a worthwhile thing on its own. However, what happens next is even cooler.

## The Prestige

![button](/public/images/posts/17DEC19/button.png){: .post-image }  

By carrying around a little state in the generated LocContext the code can now retrieve a localization key (e.g. "DisbandCompanyAction.Running") from the localized string (e.g. "My master Dumbledore is disbanding the 1st Levy..."). By caching the context for the strings that the game uses, I can later retrieve them and that enabled me to create the following magical Unity Button game object:

```c#
public class LocEditButton : UI.FPGUIBase
{
    private Button MyButton { get; set; }

    private void Awake()
    {

        MyButton = GetComponent<Button>();
        MyButton.onClick.RemoveAllListeners();
        MyButton.onClick.AddListener(() =>
        {
            var parentLabel = transform.parent.GetComponent<TextMeshProUGUI>();
            if (parentLabel == null)
            {
                MyButton.onClick.RemoveAllListeners();
                Hide();
            }

            var locContext = FPGLoc.GetContextForString(parentLabel.text);
            if (locContext != null)
            {
                FPGUI.Instance.ShowLocEditWindow(locContext);
            }
        });
    }
}
```

This simple button can be added as a child of any UI text label and it gives the user the ability to edit that string. In the editor window you can see the available tokens to be used in the string (on the left) along with their values in this particular context. You can edit the text directly in the white box and see a preview in the green box. The buttons allow you to replace the existing string, or add a new one for that key. (The localization code will use a random if it finds multiple matches)

![editor1](/public/images/posts/17DEC19/editor1.png){: .post-image }  
![editor2](/public/images/posts/17DEC19/editor2.png){: .post-image }  

## The Right Time

Part of the challenge of making a change like this is identifying the correct time to do it. Game development is a mixture of a number of pressures that are often competing. To me it seems that the choice is often between (1) making progress on the game as quickly as possible and (2) writing maintainable systems. 

I thought that this was the right time to improve this system because the game is getting to a point where the extensible architecture enables me to quickly add new actions to the game. Stopping to think about and write strings for these actions (to me) is quite hard, and nothing beats being able to iterate quickly while the game is running to see what feels best. Furthermore, it allows me to outsource the work of writing and editing these strings to other people.

Remember those 9 methods I made abstract near the beginning of this process?

> **Yuk!**

I hope you enjoyed reading about this technical aspect of the project, and that perhaps my lessons will be useful in avoiding your own pitfalls in the future. Check back later this week to find out about the crapload of stuff coming in 0.1.5.

Thanks for reading!
