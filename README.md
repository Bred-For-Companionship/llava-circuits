## Note for Mats application example code
I'm a bit confused about the intent of the questions. Is the sought after quality skill at *design of experiments* (teasing out causal links and controlling for confounders etc.) or *building experiment infrastructure* (good ML engineering)? Or both?
I wish I had code that showcases both! But I don't. I'm going to assume it's the first sklll set (experiment design). 

## Mini project intro
Circuits have been discovered in LLMs of various sizes, so it is logical to assume that they exist in VLLMs. Different from the vision-language models supported such as CLIP, VLLMs project the vision output of an image encoder (often the ViT from CLIP) into LLM space, opening up a new world of possibilities in image-grounded LLM prompting. The poster-child of this class of models is LLAVA, which I explore in this mini project.

I start with activation patching (32 layers w/ 32 heads each in the LLM backbone of LLAVA) and will explore attention patching afterwards.

### Experiment setup: The RGB game
In this task, an image with one red patch within a 24*24 grid of blue patches is provided (Llava uses 24*24 patch size) . The model is then prompted: “'Which color is the odd color out? Respond in one word'. The corrupted images consist of either a green patch in place of the red one or a blue one in its place, i.e. a solid blue canvas. LLava gets this task right easily. Interestingly, for the solid blue canvas, its answer is “white”. For solid canvases of other colors, it also chooses white as the non-existent odd color out. 

![alt text](rgb-intro.png)

It is hypothesized that heads fulfilling these roles would be involved in this task:
1.	Heads attending to ‘color’ and ‘odd one out’ to understand the task
2.	Heads that attend to the image tokens, specifically the odd tiled color, which I’ll call “Foreign medium point of interest localization heads”, and help color extractor heads (discussed below) query the odd colored tile by composing with these heads query side.
3.	A head that extracts the color attribute of the image token corresponding to the odd colored tile and dumps it into the residual stream  at the [end] token, which I’ll call “Color extractor head”

We focus on 2 and 3. The following patching results, if observed, would validate their existence::
1.	When corrupting the red tile to either green or blue, restoring the activation of the color extractor head should boost the logit score of red significantly. They are like name-mover heads from IOI in a sense.
2.	When corrupting to either green or blue, restoring the attention pattern of the color extractor head, but not its activation output, should steer the dominant logit score to green/blue, the color of the actual tile in place of the red one, as it would attend to the correct “odd tile out” and extract its present corrupted color.
3.	When corrupting to blue, restoring the activation of the foreign medium point of interest localization head(s) should not increase the logit for red, as the head’s effect is only to compose with the color extractor head, making it query the odd tile. Conversely, it should increase the logit for blue (output is “white” when there’s no odd tile), as it would write to the query of the color extractor head to make it attend to the image token where there was an odd tile out when uncorrupted.
When corrupting to green, restoring the activation of the  foreign medium point of interest localization head(s) should not change the logit distribution, as nothing was broken from its perspective. There is still an odd colored tile for it to attend to. The fact that it is green instead of red is irrelevant given the head’s role, and the patched answer should be the same as the corrupted answer- “green” (When it is blue, there is no odd tile out, though)

**Workflow**

LLava-circuits.ipynb contains the experiment setup and preliminary results. We first override the attention implementation of LLava to make it output from individual attention heads, then set up hooks to cache and patch activations. Following this, we compute logit differences as in standard activation patching setup. 

**Preliminary results**

The first graph is for red restoration when corrupting to green, the second is for red restoration when corrupting to blue, and the third is for shifting logits to blue when corrupting to blue. The first two test for color extractor heads and the last tests for point of interest localization heads. 
 
There seems to be preliminary evidence for the existence of such heads, but they need to be corroborated with other investigations. The POI localization circuit in particular may have more going on than expected. 

Head 18.6 seems to be a color extractor head, as it restores the logit to red, regardless of whether it was patched blue or green. Patching the attention pattern but not the activation for head 18.6 and seeing if it then changes the dominant logit to blue/green would corroborate its role, but I haven’t gotten to this yet. 

When trying to detect the foreign medium POI localization head (third plot) by observing heads that shift the logit to blue when patched to blue, I found three heads that all seemed to fulfill such a role, heads 14.8, 18.9, and 26.16. All these heads also do not increase the logit for red when either green or blue patching occurs, consistent with the behavior of a POI localization head. Interestingly, if the circuit operated in the hypothesized manner, the POI localization head should precede the color extractor head (layer 18) in order to compose with it query side. So why was the head at layer 26 able to shift the prediction to blue?



