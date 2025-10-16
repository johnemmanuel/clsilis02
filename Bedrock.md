Skip to content
Chat history

You said:
I developed a chat application at work using bedrock.agents. problem is, it gives me different results each time I invoke it. Temperature is.set to 0.1. The system prompt is well designed. It uses a Kendra index for the knowledgebase. How can I ensure that bedrock does not use external knowledge and only the kendra knolwedgebase. How can I make the mom read specific propertries.from Kendra such as ordercode, title etc. What am I missing? What are the best practices to get great  quality results with bedrock and kendra.
ChatGPT said:
You're running into a common challenge when combining Bedrock Agents with Kendra — even with low temperature and a strong system prompt, you’ll still see variability if:

The LLM is allowed to hallucinate or use external knowledge, or

The retrieved Kendra context is not structured or constrained enough, or

You haven’t forced the agent to only answer from retrieved documents.

✅ Best Practices to Get Consistent & High-Quality Responses from Bedrock + Kendra
1. Force the Model to Only Use Kendra — No Outside Knowledge
Include strict grounding instructions in both:

System prompt

User prompt template (instruction block before query)

For example:

“You must answer only using the context retrieved from Kendra.
If the answer is not explicitly found in the provided documents, respond with: I don’t know based on the available knowledgebase.
Do not use external knowledge or assumptions.”

Also enable "answerGeneration": { "source": "KENDRA" in your Bedrock Agent configuration (if using JSON or CloudFormation).

2. Control What Kendra Returns — Don’t Feed Raw Documents
Instead of giving the full text blob, transform each Kendra result into a structured schema before passing it to Bedrock:

[
  {
    "orderCode": "12345",
    "title": "Comprehensive Metabolic Panel",
    "description": "...",
    "specimenType": "Serum",
    "turnaroundTime": "24h"
  }
]
Then instruct the model:

“When answering, only reference properties such as orderCode, title, and turnaroundTime. Ignore any irrelevant text.”

3. Make Retrieval Deterministic
Set TopK = 1 or 2 — avoid noisy retrieval.

Use semantic + keyword hybrid ranking if available.

Ensure Kendra metadata & custom attributes are well defined (e.g., orderCode, department, category) and boost them in ranking profiles.

4. Avoid Bedrock Agent Hallucinations by “Citing”
Tell the model to quote the source for every answer:

“Always include the orderCode and title from the retrieved item as part of your response. Never invent new codes.”

5. (Optional) Wrap with a Deterministic Orchestrator Instead of Freeform Chat
Do NOT let the LLM decide how to use knowledge.

Instead:

Approach	Control Level	Variability
Plain Bedrock Agent Chat	❌ Low	🔥 High
Rag Wrapper (Retrieval → Fixed Prompt → LLM)	✅ High	❄️ Low
Tool Calling with Explicit Parameters	✅✅ Very High	❄️❄️ Very Low
🎯 Summary Checklist
Problem	Fix
Model mixing in internet knowledge	Add strict grounding rules + disable external fallback
Kendra returns noisy data	Preformat results into clean JSON before prompting
Answers differ each time	Reduce TopK, force single-source grounding
Model ignores metadata fields	Explicitly instruct it to reference named properties only
Want me to draft a complete “golden” system prompt or a sample Bedrock Agent configuration with strict grounding logic? Just paste your current prompt / setup and I’ll optimize it.

Also — are you using Bedrock Knowledge Bases or Kendra via external Lambda function? That changes how much control we have.

You said:
Architecture is as follows 

Client->Lambda->bedrock lambda->bedrock agent->lol

Agent is configured with Kendra knowledgebase. The persona is that of an expert in human disease symptoms, conditions and diagnostic tests. The agent will recommend tests based on user query and the Kendra knowledgebase. Generate a golden prompt for me please. Tell me if I am.overlooking somethimg. What do you mean by ' Tool Calling with Explicit Parameters' how is this done.
ChatGPT said:
Got it — thanks for the architecture context. You're running a Client → Lambda → Bedrock Lambda → Bedrock Agent → LLM chain, with a Kendra-backed knowledge base and a diagnostics recommendation persona.

Let’s tackle your three requests:

✅ 1. Golden System Prompt (Strict Grounded Diagnostic Assistant)
You can use this as your Bedrock Agent system prompt or inject it at the beginning of your Lambda-driven context:

🎯 Golden System Prompt – Grounded Diagnostic Assistant

You are an expert medical diagnostics assistant that recommends lab tests based only on the knowledge provided from Kendra.

Mandatory Rules:
Use only the retrieved documents from the Kendra knowledge base.

Do not use medical knowledge outside of the provided context. If no clear answer is found, respond exactly with: “I don’t know based on the available knowledgebase.”

When recommending tests, include at minimum:

orderCode

title (test name)

specimenType (if available)

turnaroundTime (if available)

Always cite the source item that was used from Kendra. Format example:
"Recommended Test: {title} (Code: {orderCode}) — based on entry '{documentTitle or ID}'".

If multiple tests apply, rank by most relevant first, and provide a brief rationale in one sentence.

Never diagnose or give treatment advice. Only recommend laboratory tests.

If User Asks Outside Scope:
If the question is not related to medical tests, symptoms, conditions, or diagnostics, respond with:
“I’m designed only to help with lab test recommendations. Please rephrase your question accordingly.”

Output Format:
When answering, always respond in this structure:

Assessment of condition/symptom: <short summary>
Recommended Tests:
1. <Title> (OrderCode: X123)
  Reason: ...

2. <Title> (OrderCode: Y456)
  Reason: ...

Source: <Kendra Document Title or ID>
Follow the above rules exactly and consistently.

Let me know if you want a slightly more friendly/human tone — this one is rigid on purpose for grounding discipline.

✅ 2. What You Might Be Overlooking
Area	Potential Gap	Fix
Kendra Result Noise	If Kendra returns top 5 docs, model may mix answers	Limit TopK = 1 or 2 and strip irrelevant text
Missing Metadata Enforcement	Agent might ignore orderCode, specimenType, etc.	Include strict output schema rules (as done above)
Agent Still Hallucinates	Bedrock may still improvise if confident	Add fallback rejection phrase: “I don’t know…”
Conversation Drift	Multi-turn conversations lose grounding	Re-inject system grounding every turn via Lambda
✅ 3. “Tool Calling with Explicit Parameters” — What I Meant
Instead of letting Bedrock freestyle everything, you expose tools/functions that the model must call with structured parameters.

For example — instead of:

"What tests should I take for fatigue?"

You instruct the LLM to call a function instead of replying directly:

{
  "name": "get_recommended_tests",
  "parameters": {
     "symptom": "fatigue"
  }
}
Your Lambda receives that, queries Kendra deterministically, formats results, and returns a structured answer.

This prevents hallucination and gives you full control over retrieval & ranking, reducing chaos dramatically. Bedrock supports this pattern similarly to OpenAI’s function calling / Anthropic’s tool use schema.

➡️ If you’re interested, I can draft a complete Agent + Function Spec JSON that enforces this mode.

🚀 Next Step
Let me know:

Do you want me to make a more conversational version of the golden prompt?

Do you want a sample Tool Schema + Lambda handler for “get_recommended_tests”?

I can generate either immediately.

You said:
Currently the llm keeps returning order code not.available even though it is .
ChatGPT said:
This is a common failure mode — the LLM is seeing the metadata (orderCode) but failing to extract it correctly, either because:

Kendra is returning unstructured context (e.g., a blob of text instead of clearly labeled fields), or

The LLM isn’t recognizing your metadata keys inside that blob, or

There are multiple formats across documents and it gives up and defaults to “not available”.

✅ Fix Strategy — Force Explicit Metadata Extraction
Instead of passing raw Kendra text, format the retrieved documents before handing them to the LLM, like this:

❌ Current Likely Format (LLM struggles to parse)
Title: Complete Blood Count
Order Code: 12345
---
Some description and random text...
✅ Recommended Structured Format
Wrap every retrieved record like this:

[
  {
    "id": "KendraDoc_001",
    "title": "Complete Blood Count",
    "orderCode": "12345",
    "description": "Used to evaluate blood cell health.",
    "specimenType": "Whole Blood",
    "turnaroundTime": "24 hours"
  }
]
Then explicitly instruct the LLM:

“When you respond, if a field is missing ONLY IF it is truly absent in the JSON above.
If a field exists, you must return it exactly as-is, without paraphrasing or reformatting.”

✅ Hardening the Prompt — Override the “Not Available” Failure
Modify your rules like this:

🚫 Forbidden Auto-Fallback:
You are not allowed to respond with “order code not available” unless you have explicitly scanned every provided item and confirmed no orderCode exists.
If at least one orderCode field exists in any retrieved record, you must use it exactly as provided.
You are not permitted to assume unavailability.

This works surprisingly well — LLMs obey rules better when you punish specific lazy behavior.

✅ Final Tip — Detect This Pattern & Auto-Correct in Lambda
Your Lambda can filter the LLM response:

If output contains "order code not available" but the retrieved context included one, reject and re-ask the LLM:

"You incorrectly responded that orderCode is unavailable. The orderCode exists: 12345. Try again."

(Yes — LLMs are obedient when scolded.)

Want Me to Generate:
✅ A full structured Kendra → LLM context wrapper template?

✅ A revised system prompt with “no fake 'not available' allowed” rules baked in?

✅ Lambda code snippet that parses Kendra → JSON → passes to Bedrock?

Just say the word and I’ll drop it in clean format.

You said:
Please generate the golden prompt again for my reference. Use a professional tone while.being polite. Ensure Guard rails are in the prompt.
ChatGPT said:
Sure — here is a refined “golden prompt” written in a professional yet approachable tone with explicit guardrails to enforce structured, grounded, and safe responses.

✅ Golden System Prompt for Bedrock Agent (Medical Test Recommendation with Kendra Grounding)
You are a Medical Diagnostics Assistant specialized in recommending laboratory tests based on user-described symptoms or medical conditions.
You must operate strictly as a test recommendation engine, not as a diagnostician or treatment advisor.

Core Operating Rules (Follow These Precisely)
Use only the information retrieved from the Kendra knowledge base to generate recommendations.

You are not permitted to rely on external medical knowledge or assumptions.

If the provided knowledgebase does not contain sufficient information, respond exactly with:
“I don’t know based on the available knowledgebase.”

When recommending tests, extract and present the following fields exactly as provided in the retrieved data:

orderCode

title (test name)

specimenType (if available)

turnaroundTime (if available)

Never fabricate or alter values.

If a field is missing, leave it blank or mark it as “Not provided in knowledgebase”.

Do not respond with “Not available” unless no record contains that field.

Always cite the source document used for your recommendation.

Use this format:
"Source: <Document ID or Title>"

If multiple relevant tests are found, list them in descending order of relevance and provide a brief, factual rationale for each.

Reject any request outside scope.

If the query involves diagnosis, treatment, lifestyle, or unrelated topics, reply with:
“I am designed only to recommend diagnostic laboratory tests. Please rephrase your question accordingly.”

Response Format (Mandatory)
Assessment: <brief summary of the implied condition or symptom>

Recommended Tests:
1. <Title> (OrderCode: XXXX)
  Specimen: <specimenType or "Not provided in knowledgebase">
  Turnaround Time: <turnaroundTime or "Not provided in knowledgebase">
  Reason: <short rationale>

Source: <Document ID or Title>
Ethical & Safety Requirements
Do not provide medical diagnosis, treatment recommendations, medication advice, or emergency instructions.

If the user indicates a medical emergency, respond with:
“Please seek immediate medical attention from a qualified healthcare provider.”

Follow all instructions above consistently and without deviation.

Let me know if you want:

A more conversational tone

GDPR / HIPAA compliance clauses

A version optimized for multiple-turn conversations

Happy to adjust!

You said:
Please optimize for multiple turn conversations
ChatGPT said:
Absolutely. To make the golden prompt work reliably across multi-turn conversations, we need to:

Maintain context across turns without drifting off-topic

Ensure the assistant can remember previous symptoms/conditions mentioned, but still only answer using Kendra-sourced data

Reinforce guardrails repeatedly so later turns don’t break safety rules

Here’s the optimized version:

✅ Golden System Prompt — Multi-Turn Optimized
You are an AI Medical Diagnostics Assistant. Your sole purpose is to recommend laboratory tests based on user-described symptoms or medical conditions, using only the information retrieved from the Kendra knowledge base.

✅ Multi-Turn Conversation Rules
Maintain context across turns, but re-evaluate new information carefully.

If the user refers to something mentioned earlier (e.g., “What about for fatigue instead?”), continue the conversation intelligently.

If there is ambiguity, ask brief, focused follow-up questions — but only when absolutely necessary for accurate test recommendation.

For every turn, apply the same grounding rules:

Use only Kendra-provided data — do not use external medical knowledge.

If no matching information exists in the knowledge base, respond exactly with:
“I don’t know based on the available knowledgebase.”

If the conversation drifts away from laboratory test recommendations, politely redirect with:
“I am designed only to recommend diagnostic laboratory tests. Please rephrase your question accordingly.”

✅ Recommendation Formatting Rules (Every Turn)
When recommending tests, extract and present the following fields exactly as retrieved from Kendra:

orderCode

title

specimenType (if available)

turnaroundTime (if available)

Never fabricate or modify values.
If a field is absent, use “Not provided in knowledgebase” — do not assume or infer.

Always cite the source document, e.g.:

Source: <Document ID or Title>

✅ Response Format (Mandatory for Every Recommendation)
Assessment: <brief restatement of the symptom or condition>

Recommended Tests:
1. <Title> (OrderCode: XXXX)
   Specimen: <specimenType or "Not provided in knowledgebase">
   Turnaround Time: <turnaroundTime or "Not provided in knowledgebase">
   Reason: <one-sentence rationale>

Source: <Document ID or Title>
✅ Safety & Ethics Rules
Do not provide diagnosis, treatment, medication, or lifestyle advice.

If the user expresses an emergency or severe condition, respond with:
“Please seek immediate medical attention from a qualified healthcare provider.”

Follow these instructions consistently for every message in this conversation.

Would you like me to:

✅ Add context summarization rules (e.g., “Summarize conversation state internally every turn”)?
✅ Provide a separate prompt for the User Prompt Template (sometimes required in Bedrock Agents config)?

Just say the word and I’ll generate it.

You said:
Please generate a user prompt.template
ChatGPT said:
Sure! Here's a User Prompt Template you can attach before forwarding the user query to Bedrock (via Lambda). This should be inserted as a prefix instruction to every user input — ensuring the LLM interprets their query in context of the diagnostics task.

✅ User Prompt Template (for Bedrock Agent / Lambda Pre-Processing)
You are receiving a user question that may describe symptoms, medical concerns, or specific lab test inquiries.

Your task is to interpret the query strictly as a request for diagnostic laboratory test recommendations.

- If the user mentions symptoms or a possible condition, determine which tests from the knowledgebase may be relevant.
- If the user mentions multiple symptoms, evaluate them collectively.
- If the request is unclear or too broad, ask one concise clarification question before recommending tests.
- If the user is continuing a previous turn, incorporate prior context unless it conflicts with the new message.

Now process the user input below and recommend appropriate tests according to the system rules.

User Query:
{{user_input}}
✅ How to Use
Inject this before user input on every request sent to Bedrock.

Do not display this to the user — it’s strictly for LLM alignment.

Replace {{user_input}} with the actual query.

✅ Optional Variant: With Previous Context
If you're carrying forward conversation history, use this version instead:

Conversation Context (previous turns):
{{conversation_history}}

Current User Query:
{{user_input}}

Interpret this as a request for laboratory test recommendations. Follow the system rules and respond accordingly.
Would you like me to:

✅ Provide this wrapped in a JSON structure formatted for Bedrock Agent configuration?

✅ Generate a Lambda helper function to auto-build this prompt per turn?

Let me know and I’ll drop it in!

You said:
Pls generate  sample Tool Schema + Lambda handler for “get_recommended_tests”?
ChatGPT said:
Absolutely! Below is a sample Tool Schema and a Lambda handler implementation that demonstrates structured tool calling for a function named get_recommended_tests. This setup forces the LLM to respond deterministically by returning structured function calls instead of freeform text — giving you full control over how Kendra is queried and how results are formatted.

✅ Tool Schema (JSON Definition for Function Calling)
{
  "functions": [
    {
      "name": "get_recommended_tests",
      "description": "Retrieve relevant lab tests from the knowledge base based on a medical symptom or condition.",
      "parameters": {
        "type": "object",
        "properties": {
          "symptom": {
            "type": "string",
            "description": "The primary medical symptom or condition for which lab tests are being requested."
          },
          "age": {
            "type": "integer",
            "description": "Optional. Age of the patient if relevant for test recommendation."
          },
          "gender": {
            "type": "string",
            "enum": ["male", "female", "other", "unknown"],
            "description": "Optional. Patient's gender if it influences test decisions."
          }
        },
        "required": ["symptom"]
      }
    }
  ]
}
✅ Lambda Handler (Node.js Example — Python version available if you prefer)
This handler:

Receives the LLM function call

Queries Kendra based on the provided symptom

Formats results into structured output

Returns final answer to the caller

import { Kendra } from "@aws-sdk/client-kendra";

const kendra = new Kendra();

export const handler = async (event) => {
  console.log("Incoming Event:", JSON.stringify(event));

  const toolRequest = event.toolInvocation;
  if (!toolRequest || toolRequest.name !== "get_recommended_tests") {
    return { error: "Invalid tool invocation" };
  }

  const { symptom } = toolRequest.parameters;

  // Query Kendra
  const response = await kendra.retrieve({
    IndexId: process.env.KENDRA_INDEX_ID,
    QueryText: symptom,
    PageSize: 3
  });

  if (!response.ResultItems || response.ResultItems.length === 0) {
    return {
      message: `No relevant test information found in the knowledgebase for '${symptom}'.`
    };
  }

  // Format structured output
  const recommendations = response.ResultItems.map(item => {
    const metadata = item.DocumentAttributes?.reduce((acc, attr) => {
      acc[attr.Key] = attr.Value?.StringValue || attr.Value?.StringListValue?.[0];
      return acc;
    }, {}) || {};

    return {
      title: metadata.title || "Not provided in knowledgebase",
      orderCode: metadata.orderCode || "Not provided in knowledgebase",
      specimenType: metadata.specimenType || "Not provided in knowledgebase",
      turnaroundTime: metadata.turnaroundTime || "Not provided in knowledgebase",
      source: item.DocumentId
    };
  });

  return {
    toolResponse: {
      name: "get_recommended_tests",
      content: recommendations
    }
  };
};
✅ Example Final Output Returned to LLM
{
  "toolResponse": {
    "name": "get_recommended_tests",
    "content": [
      {
        "title": "Complete Blood Count",
        "orderCode": "12345",
        "specimenType": "Whole Blood",
        "turnaroundTime": "24h",
        "source": "doc-001"
      }
    ]
  }
}
✅ Next Steps
Would you like me to:

🔹 Generate a Python version of the Lambda handler?

🔹 Show how to wire this tool schema into Bedrock Agent config?

🔹 Wrap the final output in the golden response format automatically?

Just say the word and I’ll expand it.

You said:
Python version please
ChatGPT said:
Sure! Here's the Python version of the Lambda handler for the get_recommended_tests tool.

✅ Python Lambda Handler for get_recommended_tests
import os
import json
import boto3

kendra = boto3.client("kendra")

def handler(event, context):
    print("Incoming event:", json.dumps(event))

    # 1. Extract tool invocation details
    tool_request = event.get("toolInvocation")
    if not tool_request or tool_request.get("name") != "get_recommended_tests":
        return {
            "error": "Invalid tool invocation or missing 'get_recommended_tests'"
        }

    params = tool_request.get("parameters", {})
    symptom = params.get("symptom")

    if not symptom:
        return {"error": "Missing required parameter: symptom"}

    # 2. Query Kendra
    try:
        response = kendra.retrieve(
            IndexId=os.environ["KENDRA_INDEX_ID"],
            QueryText=symptom,
            PageSize=3
        )
    except Exception as e:
        print("Kendra query failed:", str(e))
        return {"error": "Kendra query failed"}

    result_items = response.get("ResultItems", [])
    if not result_items:
        return {
            "message": f"No relevant test information found in the knowledgebase for '{symptom}'."
        }

    # 3. Format structured response
    recommendations = []
    for item in result_items:
        metadata = {}
        for attr in item.get("DocumentAttributes", []):
            key = attr.get("Key")
            val = attr.get("Value", {})
            metadata[key] = (
                val.get("StringValue") or
                (val.get("StringListValue")[0] if val.get("StringListValue") else None)
            )

        recommendations.append({
            "title": metadata.get("title", "Not provided in knowledgebase"),
            "orderCode": metadata.get("orderCode", "Not provided in knowledgebase"),
            "specimenType": metadata.get("specimenType", "Not provided in knowledgebase"),
            "turnaroundTime": metadata.get("turnaroundTime", "Not provided in knowledgebase"),
            "source": item.get("DocumentId", "Unknown")
        })

    # 4. Return deterministic tool response
    return {
        "toolResponse": {
            "name": "get_recommended_tests",
            "content": recommendations
        }
    }
✅ What’s Next?
Would you like me to:

🔹 Include validation to detect when LLM claims “orderCode not available” incorrectly?

🔹 Provide a wrapper that auto-converts this into the structured response format from your golden prompt?

🔹 Show how to connect this Lambda into Bedrock Agent's tool definition flow?

Just say and I’ll extend it!

You said:
connect this Lambda into Bedrock Agent's tool definition flow? Also generate the wrapper that auto converts tool response into structed.response 
ChatGPT said:
Absolutely! Let’s finish the loop by:

✅ 1. Connecting Your Lambda Tool (get_recommended_tests) to Bedrock Agent
When defining your Bedrock Agent, you must register your Lambda as a Tool. In AWS, this is typically done via CloudFormation, AWS Console, or SDK.

Here’s an example Bedrock Agent Tool Definition in JSON (can be used via CLI / SDK or CloudFormation):

{
  "agentTools": [
    {
      "name": "get_recommended_tests",
      "description": "Retrieve relevant lab tests from the knowledge base based on a medical symptom or condition.",
      "type": "LAMBDA",
      "lambda": {
        "arn": "arn:aws:lambda:us-east-1:<ACCOUNT_ID>:function:get_recommended_tests"
      },
      "inputSchema": {
        "json": {
          "type": "object",
          "properties": {
            "symptom": { "type": "string" },
            "age": { "type": "integer" },
            "gender": { "type": "string", "enum": ["male", "female", "other", "unknown"] }
          },
          "required": ["symptom"]
        }
      }
    }
  ]
}
Once this tool is registered, Bedrock can automatically call your Lambda when the LLM decides to use get_recommended_tests.

✅ 2. Wrapper: Convert Lambda Tool Response → Structured Final Answer (Golden Prompt Format)
The tool currently returns:

{
  "toolResponse": {
    "name": "get_recommended_tests",
    "content": [ ... list of recommendations ... ]
  }
}
Now we wrap it so the LLM never freestyles formatting. We control it using this wrapper:

def format_final_response(recommendations, symptom):
    if not recommendations:
        return f"I don’t know based on the available knowledgebase."

    response_lines = [
        f"Assessment: Possible evaluation for '{symptom}'",
        "",
        "Recommended Tests:"
    ]

    for idx, rec in enumerate(recommendations, start=1):
        response_lines.append(
            f"{idx}. {rec.get('title')} (OrderCode: {rec.get('orderCode')})"
        )
        response_lines.append(f"   Specimen: {rec.get('specimenType')}")
        response_lines.append(f"   Turnaround Time: {rec.get('turnaroundTime')}")
        response_lines.append(f"   Source: {rec.get('source')}")
        response_lines.append("")  # Blank line between entries

    return "\n".join(response_lines)
Then modify your Lambda handler like this (final section):

# 4. Wrap final output into golden prompt format
from_formatter = format_final_response(recommendations, symptom)

return {
    "toolResponse": {
        "name": "get_recommended_tests",
        "content": from_formatter
    }
}
✅ 3. Full Flow Summary
Step	Component	Responsibility
User Ask	"What tests for fatigue?"	—
Bedrock Agent	Decides to call tool → get_recommended_tests	✅ Tool Invocation
Lambda	Queries Kendra → formats recommendations	✅ Data Layer
Wrapper	Enforces Golden Prompt Format	✅ Output Discipline
Bedrock Returns	Deterministic structured answer	🎯 No Hallucination
✅ Want Me To…
Generate a CloudFormation-ready full Agent definition with the tool wired in?

Add logging + error fallback logic?

Provide a multi-turn memory guard so symptom context persists across turns?

Just say “Do 1” or “Do 1 and 2” etc — and I’ll deliver!

You said:
How do we determine symptoms. The user may not always specify them.
ChatGPT said:
Great question — and very important. In real conversations, users rarely say "My symptom is X." Instead, they say:

"I feel tired all the time."

"My joints hurt when I wake up."

"Should I get my sugar levels checked?"

"What blood work do I need before surgery?"

So your system must infer the symptom or condition from free-text user input, even when the user hasn’t explicitly labeled it as a “symptom.”

✅ Strategy: Extract or Infer the Symptom Before Tool Invocation
There are three escalating strategies, depending on how strict you want to be:

Approach	Pros	Cons
1. Infer symptom directly from user text using LLM classification	Fully automatic, no user interruption	Might misinterpret occasionally
2. Ask follow-up question only when unclear	More accurate	Adds latency
3. Hybrid: Try inference → If low confidence, ask confirmatory question	Best balance	Requires confidence scoring
✅ Example: Symptom Extraction Prompt (to run before calling the tool)
You can insert this as a pre-processing LLM step:

“Extract the main symptom or medical condition from the following user message. Respond with only one of the following formats:

SYMPTOM: <extracted symptom> → if clear

AMBIGUOUS → if no clear symptom can be determined”

Input:
"My hands feel tingly and cold sometimes. I also get headaches."

LLM Output (ideal):
SYMPTOM: tingling hands

✅ Example: If Ambiguous → Ask a Clarifying Question
If the model responds AMBIGUOUS, then send back:

“I can help recommend lab tests, but I need to understand better. Are you asking about headaches, tingling, or something else?”

✅ Plugging Into Your Flow
User → LLM (Symptom Extractor) → 
   → If SYMPTOM:X → Invoke get_recommended_tests(symptom=X)  
   → If AMBIGUOUS → Ask clarifying question → Loop  
This can be implemented before Bedrock Agent tool-calling logic OR as part of your Bedrock Agent instructions.

✅ Want Me To Generate:
A ready-to-use "Symptom Extractor" prompt block?

A Python helper function to auto-run extraction before invoking tools?

An updated Tool Schema that includes symptom_source metadata (e.g., direct / inferred / confirmed)?

Just say “Give me 1 & 2” or “All three” and I’ll deliver!


Responses will use another model and apps will be unavailable until your limit resets after 1:48 PM.

Upgrade to Go



No file chosenNo file chosen
ChatGPT can make mistakes. Check important info. See Cookie Preferences.
