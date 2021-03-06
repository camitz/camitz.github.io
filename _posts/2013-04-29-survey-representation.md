---
date: 2013-04-29 09:34
title: On representing a survey with XML
Type: post
tags: [survey, xml, specification, standard, flamingo]
categories: [Tech]
---

With my new survey software suite, a survey is as easy to enter as

	What did you drink last night?
	o Gin/Tonic
	o Marguerita
	o Cosmopolitan
	o Other: []

My clients will be able to enter survey with ease and mostly without my involvment. They will be able to share survey by email for review.

Now all I have to do is get it out of my brain and into a compiler. But why not?

I'm picturing something like markdown which outputs an intermediary XML representation in an update-as-you-type-editor, maybe online. With some meta data in the header you'd be able to publish it in a dropbox folder just as I am this blog post. 

It's the representation at the core of this that I want to talk about today. I propose a new representation of a survey. I'm calling it **Flamingo**.

## Background

I sometimes dabble with surveys in particular [RDS](https://www.researchgate.net/publication/233419485_Implementation_of_Web-Based_Respondent-Driven_Sampling_among_Men_Who_Have_Sex_with_Men_in_Vietnam) surveys. I made my own survey engine many years back. It was replaced by the research team with a modified version of LimeSurvey. 

LimeSurvey is very nice but it's stuck in an inflexible model. One it shares with many other engines and XML representations I've seen ([1](http://quexml.sourceforge.net/node/1),[2](http://www.triple-s.org/sssxml1.htm),[3](http://manual.limesurvey.org/wiki/Importing_a_survey_structure),[4](http://www.ncbi.nlm.nih.gov/pmc/articles/PMC2244339/)). Sorry for ranting about LimeSurvey, it just happens to be in the line of fire.

There are the database tables. Questions. Answers. Question attributes. Conditions. 

There are the question types. Radio list. Dropdowns. Yes/no. Text. Numeric. LimeSurvey offers several different types of array questions. 20 or so different types altogether. LimeSurvey survey calls this "vast". Nonetheless, I've had to add three question types and hack five. 

So there's one motivation for rethinking the whole thing. I do not propose a presentation enginge that we will never need recoding for different needs and purposes but I most certainly want to separate the presentation from survey content and logic.

This last week I've been dealing with some pretty complex stuff involving a country selector. We ultimately had to find LimeSurvey-conforming solutions to it. Hacking the model and presentation was too big an operation.

What I propose is not only a brand new representation of a survey but a whole new way of thinking about a survey from the bottom up, including presentation and interpretation of responses.

The key advantages are:

- High flexibility
- Partial compliance with HTML
- Minimum dependence between survey, response, presentation and interpretation of responses
- Redundant referencing of survey elements
- Cross references of survey elements
- Repetition, validation and other survey logic included
- Dynamically generated survey elements

## Start

Let's give it a go then. Let's go way back. What's the atomic building block of a survey? The option. I propose this is a survey.

	<option>Gin/Tonic</option>
	<option>Marguerita</option>
	<option>Cosmopolitan</option>
	<option>Steak</option>
	<option>Lobster</option>
	<option>All of the above</option>

That's completely answerable. The questionnaire is interpretable and so are the responses, given the right context which may be given as a title or verbally for all I know. No assumptions are made, particularly about which or how many choices are possible together nor what question is actually being asked. The response recorded may be just a table with entries identifying the user and the options selected.


Just for the sake of it, let's impose some implicit logic and provide the user with some context.

    <group>
        <label>What did you drink last night?</label>
        <option>Gin/Tonic</option>
        <option>Marguerita</option>
        <option>Cosmopolitan</option>
    </group>
    <group>
        <label>What did you eat last night?</label>
        <option>Steak</option>
        <option>Lobster</option>
        <option>All of the above</option>
    </group>

This is a refreshing change of perspective for me. All these years I've been thinking of a survey as a list of questions. What I've come to realize it that it's really a list of options. The question is just meta data and so is the grouping and any explicit or implicit logic.

This is my basic proposition for a new way of representing a survey - with XML, using two basic elements, options and groups. 

Before we move on, though, let's change the name of those two elements. The thing is, compliance with HTML, turns out to be a good idea. It makes building a presentation engine based on this scheme much easier. One, most of the stuff you can display straight off. Two, for those knowledgeable in HTML, the documentation is mostly implicit. 

I've come up with a set of principles that seem to work rather well for me. Partial compliance with HTML is the second. Hence I'm calling groups *div* and options *input*. Ignore the fact that inputs, if they really were in HTML, would have no content. Ignore also the fact input elements without any other attributes produce textboxes.

## Basic principles

The complete set of basic principles for Flamingo is as follows.

- Two atomic elements, input and div, that largely define a questionnaire
- Comply with HTML when possible
- Always assume multiple choice options
- All the elements are unquiquely identifiable by their XPATH
- All contents of label attributes and elements act as resource keys when

Incidendtally, the second to last principle is implicit in XML.

We've covered two of these principles, although their merit may be unclear for the moment. Let's move on to the third.

## Assume multiple choice

The default input is a checkbox. Heh? Bear with me, I just want to change the default. 

Your presentation engine's first point of order is hence to replace

	<input>Gin/Tonic</input>

with

    <input type="checkbox"><label>Gin/Tonic</label>

Not so long ago form elements came in three flavors, text (including text area), single choice (radios and dropdowns), and multiple choice (checkboxes). This shaped our thinking of what a survey was and what it could do. **In effect it is constraining research.** HTML5 specifies a wealth of new form elements and we've been introduced to many more in the form or custom widgets. Rather than expand the model significantly (infinitely) the proposition is to narrow it down to the basics.

In my version, the basic questionnaire is one built from one basic element, the option. And you can select anything! That means checkboxes. Constraints and validation come on top of this. It's context. It could easily be provided by someone looking over your shoulder and correcting you. I'm not saying it should be that way, but that's the basic starting point

Now let's break the mold.

	<label>What did you have for dinner last night?</label>
	<div>
	    <label>Drink:</label>
		<input>Gin/Tonic</input>
		<input>Marguerita</input>
		<input>Cosmopolitan</input>
		<input>Tequila sunrise</input>
	</div>
	<div>
	    <label>Eat:</label>
		<input>Steak</input>
		<input>Lobster</input>
		<input>I don't know</input>
	</div>
	
What are those? Questions? Subquestions? Any survey model that relies on question as the basic atomic element will run into an "oops" here. I could probably hack this specific layout into LimeSurvey without altering the model, but for each variation I'd be doing more hacking. And for LimeSurvey that means php. Isch.

Easy for me to say, I'm specifying the representation. You're coding the presentation engine and I'm saying you have to be super adaptive. But I'm going to make life as easy as I can for you. Be like your browser. It can present infinite variations of HTML, even faulty HTML. I'll partially comply with HTML so you can display most of everything minimum of regular expression text manipulation. And since the input is the basic atomic element, what you send back as a response to the server will be just those. If your engine can't display the survey exactly the way the author intended, resort to unformatted HTML straight off (just remember to replace the inputs appropriately) and post the responses as is.

Apparently time to show you what a response might look like.

## Responses and the XPATH

The response database table might look like this. It's a recommendation and not really part of the specification I propose.

    respondent timestamp xpath NPATH   value    content
    1   123456  "div[0]/input[3]"    "div[0]/input[3]"    ""  "Tequila sunrise"
    1   123556  "div[1]/input[0]"    "div[1]/input[0]"    ""  "Steak"
    1   123756  "div[1]/input[1]"    "div[1]/input[1]"    ""  "Lobster"


The complete XPATH is noted along with the value. The value may be a textbox value or something from a custom widjet. This means the interpretation of response is dependent on the order and layout of the survey. A concern for the results interpretation, in my opinion. However, if you need more control, add a *name* attribute to the input element. The last entry above may now read:

    1   123756  "div[1]/input[1]"    "div[1]/input[@name='lobster']"    ""    "Lobster"

That's NPATH, with the n for name. Using either the XPATH or the NPATH you can reference the element in your survey XML and the rendered questionnaire. Also key. It helps in server side validation and cross referencing at the interpretation stage.

But wasn't the objective to reduce dependency between response and representation? Yes. Firstly, the right most column is still there, with the content. It may be all you need. The additional columns are there to aid you in the interpretation.

Secondly, while the response and interpretation of the response is dependent on the survey layout, the thing to note, is that the response model is not. This is key.

This is one of the things I address with the XPATH/NPATH/value/content redundancy. When you alter the survey you will be able to compare the XPATH with the NPATH, back trace through the response table and be able to deduce what option is chosen. To aid you have the complete version history of the survey, don't you?

Reorder, add or delete questions, divide or merge questions, duplicate the question... Have two versions of the survey running in parallell. You can design a response server that will be able to record data in a way you deem sufficient to interpret.

I don't consider this more than a recommendation. Exactly what you choose to record is entirely up to you. You may even submit an xml representation of the response, if you wish.

Timestamp? Again a recommendation. This is just where we are today regarding what data is. The age when a response was a check in a box is over. Valuable information can be gleaned from the behavior of the respondent, including how long he/she takes to respond and how many times the answer is changed. The least information is a cross section of the data including only the latest answer.

One final thing to note, the engine may reorder everything as you wish. Maybe the author has requested randomized questions. This should not affect the XPATH of the options. The XPATH references the survey XML template, not the rendered display. That includes dynamically generated elements which we'll get to later.

### Another example

To further the discussion with another example, consider the following. I've introduced that label short hand here, but that's not the point.

	<div>
	    <label>Rate the following dishes</label>
	    <div label="Steak">
	        <input>Hate it</input>
	        <input>It's ok</input>
	        <input>Love it</input>
	    </div>
	    <div label="Lobster">
	        <input>Hate it</input>
	        <input>It's ok</input>
	        <input>Love it</input>
	    </div>
	</div>
	
It should be apparent how this is intended to be displayed. LimeSurvey designates these *array questions* and we've all seen them in customer surveys, rating product experience, for example. It's very similar to the former example except that the content of the input elements repeat. The options' content are to be displayed as column headers. Lobster and Steak are row headers. 

However, if the presentation engine does not have support for this particular layout, that's fine too. Just display as is (formatting the inputs, obviously). The survey is entirely legible in either case.

## Single choice and multiple choice

As I've said, I consider all input elements checkboxes by default, because that's what I consider a survey, options. If you're in Europe, both steak and lobster may not be acceptable at the same time. I consider it validation and validation can be specified in my XML schema but the basic assumption is that it is not required. It's meta data, same as questions and part of a context which may be given in XML or verbally or be implicit.

So you may parse type="checkbox" (your default as opposed to "text"), type="text", type="radio". I consider that validation also and so does apparently the HTML5 group since they provide additionally type="email", type="date" etc. 

Consider radio buttons. In order for them to work exclusively, they must have the same name. That implies that the following are alike from the presentation engine stand point.

    <input name="eat">Lobster</input>
    <input name="eat">Steak</input>
and 

    <input type="radio">Lobster</input>
    <input type="radio">Steak</input>

We've indicated single choice. There are subtle differences, though. The first may produce a dropdown or any other single choice input method of your devising while the second definitely hints radio buttons. The response recorded may also be different.

    1   123756  "input[0]"    "input[@name='eat']"    ""    "Lobster"

vs

    1   123756  "input[0]"    "input[0]"    ""    "Lobster"

See how beautifully that works? By column 3, they are equivalent.

I'll also allow:

    <div type="radio">
        <input>Lobster</input>
        <input>Steak</input>
    </div>

and

    <div max="1">
        <input>Lobster</input>
        <input>Steak</input>
    </div>

What about five-to-ten-tuple choice questions? Single choice is just a subset of multiple choice. The constrained mindset we're used to stems from HTML 1.0, or even further back, an antique radio with preset stations. That's why they're called radio buttons, in case you've ever wondered.

Imagine a set of attributes for your steak. Bloody through charred. Salted. White pepper. Black pepper. T-bone or rib eye. Some are exclusive, some are not. With LimeSurvey you would be forced to design your survey in a restrictive way, dividing up into several questions. I'll give you much more flexibility.

That's why I prefer the last example to be honest, the one with max. But even multiple choice is just a subset of survey logic which I'll get to in a moment.

In an above example there was a semantically different checkbox: "I don't know". It's not clear whether this should  be a radio button or check box. At the basic level, if the presentation has no other hints, I consider a concern for response interpretation.

Technically though, there are many ways to infer abstention and ignorance. I may wish to have one "I don't know" and one "I don't want to answer". Or I may combine them. They may be placed before or after the "Other" option. I think this should always be up to the author.

### Textboxes 

What about textual data, keyboard entered data in textboxes? Again, I consider keyboard entered data just another option. Consider:

<label>What did you wear?</label>
<input/>

All browsers display that as a textbox.

The response may be recorded as:

    1   123456  "option[0]"    "option[0]"    ""  "My dad's pink tux"

Anyone who sees this out of context will not know whether this was a checkbox or a textbox or anything else. He or she may or may not be keen on knowing, but either way, it's not a basic premise.

As mentioned, feel free to use type="email" or anything else supported by HTML5 if you wish. It's part of this spec too and it hints the engine of what we expect in terms of display and validation.

## Survey logic

Survey logic encapsulates part of what we've discussed above. I consider three separate parts.

- Constraints/Validation
- Control
- Display


### Constraints/Validation

Constraints usually come in to play for specifying how many checkboxes we can select in a group, what type of characters we may enter into a field. This is closely related to validation in my view. I may convey information of what I expect of the user experience. Or I may leave it to you just get me reponses in a way of your design just so long as they fit my validation. That could mean using radio buttons. Or it could mean a flashing red alert explaining to the user that one too many options has been selected. 

The max attribute, and any other that HTML5 specifies, is a shorthand of the constraint attribute. The following is equivalent.

    <div constrain=".option.count('checked')<=1"/>

The following is just slightly more rigid. It implies that the constraint is to be executed at validation time though does not specify when the validation time might be. It may be on click.

    <div validate=".option.count('checked')<=1"/>

I may provide the following information if I'm in the mood, employing another HTML attribute. This is a more hands on constraint.

    <input onchange="if(this.checked){//div[@name='cook'].unset('checked');this.set('checked')}">Medium rare</input>

It's supposed to uncheck all other cooking options. It is not javascript and makes no pretense to be either. You'll have to parse the XPATH, for starters, loop through the resulting nodes and add appropriate identifiers. For security reasons, no presentation will want to execute javascript from an xml sheet, anyway.

### Display

Similarly, the enabled attribute, used to control where an option is visible/enabled, already implies a condition, for example:

    <input enabled="!(//option[@name='steak'].checked)">Medium rare</input>


### Control

Finally, control logic is about what questions are executed when and possibly in what order, depending on responses and other circumstances. It should be clear by now that I'm not about to introduce a goto statement. I think control concerns should be mapped to display logic as far as possible. Having said that, I haven't ruled out control logic completely though I can think of no circumstances off hand where it might come in handy.


## Other features

### Cross references

To repeat a group or option anywhere (even before the definition), just reference.

    <input ref="input[3]"/>

will suffice if it's unique. Otherwise maybe our named lobster.

    <input ref="@name='lobster'"/>

We can repeat that as many times as we want. The XPATH makes sure we can interpret the response. This makes it easy to have several questions with long list of countries. In fact, one of the things that prompted these ideas was the use of long lists in LimeSurvey. If multiple options are permissible then LimeSurvey will dynamically create a response table with as many columns for that question as there are options. You can quickly run into the limit of number of columns allowed by MySQL and before that, php stalls.

My take on the issue.

    <div label="What countries did you visit in 2010?" ref="@name='country'"/>
    <div label="What countries did you visit in 2011?" ref="@name='country'"/>
    <div label="What countries did you visit in 2012?" ref="@name='country'"/>

The concern for the presentation is to decide on a multiple choice dropdown, a full page of checkboxes with an arbitrary number of columns or perhaps an object store backed combo box widget.

### Dynamically generated survey elements

Cross references together with the severing of the dependency between responses and survey implies the following rather significant bonus. The questionnaire may be dynamically constructed. Arbitrarily repeated questions are a breeze. In fact there is little need for a representation at all for simple ad hoc questionnaires. Just hard code a form and send it off to your response engine. (That's rather stating the obvious. What I mean is, do try on my recommendation for a response table.)

Take the previous example with multiple choice countries. When the user selects a country, let the presentation generate a new dropdown. This is a real life example which I've had to solve with LimeSurvey in a backwards bending way the previous week.

Remember though, implicitly, you're adding elements to the template, not to the rendered HTML. It may or may not have implications for the XPATH and hence the responses. That's the rule anyway.

However, dynamically generated survey elements is more than just a bonus, it's actually implied in many cases by the principle of always assuming multiple choice. Consider the following.

    <div label="Eat">
        <input>Steak</input>
        <input>Lobster</input>
        <input repeat="true" label="Other"/>
    </div>

The engine should generate the last input as many times as the user requires. To be honest I think this is an annoying loop hole, one of many to be found. I'd sooner let the author opt in via the repetition feature described later. But I'd like you to give me a chance to find a way that would not break my list of principles or to patch it up.

## Presentation hints

I want to have dependencies between the representation and the presentation as weak as possible. But hints to the presentation has obvious benefits, for example, indicating preference for radio buttons over dropdown, page breaks and instructions. 

The class attribute seems the immediate choice for presentation. That way a tip can be italicized by CSS with minimum fuss.

    <label class="tip">
    
    <div class="indent3">

    <div class="dropdown">

    <label class="faq">Why are we so interested in what you had for dinner?</label>

I intend to specify a recommended list of such hints. Presentation engines are free to specify additions to that list as long as they commit to displaying surveys that don't conform to them.

## Resource keys

That last one deserves an extra look. We never got around to the last principle, the one about resource keys, did we? That's what this is.

All contents of label attributes and elements act as resource keys. That smacks primarily two flys, images and locale. I'm just mentioning it, I think most can see how this will work out. The basic idea for languages is that you design the survey in your prefered language. Then you simply extract the indexes and write a translation in any other language you require. 

If you write a FAQ, then the above indexes into that. The label may be just a question mark icon. Up to you.

## Repetition

Repeating the country dropdown based on the answer given to a previous question, try:

    <div ref="@name='country'" repeat="//option[@name='nCountriesVisited'][0].value" />

Addressing the problem mentioned earlier, when selecting an option generates new option, you would write for instance, given they are single choice dropdowns:

    <div ref="@name='country'" repeat="..//option.count('checked')+1" />

I'll be the first to agree, that's a challenge to implement, but I suppose, if the history of HTML compliance is any guide, I suppose I could allow for partial compatibility.

However, the above is bound to happen quite often so the spec will probably accept a special case.

    <div ref="@name='country'" repeat="true"/>

That may clue to a way out of the dilemma mentioned earlier, with the "Other" option. If you really wanted a repeating textbox, then something like this would accomplish it.

    <div>
        <label>Eat:</label>
        <input>Steak</input>
        <input>Lobster</input>
        <input repeat="true" label="Other"/>
    </div>

## Closing

Thank you for reading. Tell me what you think. Consider the spec open source, work in progress and feel free to contribute. I'm going to put up a wiki. 

It's a long way of to 1.0. The markdown type app, a presentation engine to work client side, a response server and a survey backend, should be developed in parallel. A little bit down the road, a UI. 

Currently I don't have the resources but I think it would be great. Consider funding me if you stand to benefit.


