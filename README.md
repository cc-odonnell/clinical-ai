# Validating Clinical Data for Quality Measures using AI

Today I’m going to walk through a tool built to automate clinical quality validation for HEDIS measures. I contributed to the enhancement of the prototype, expanded core logic and SQL pipelines, authored most of the documentation, and designed the future, scaled architecture in a GCP environment. 

You may know that HEDIS measures are widely used to assess quality in healthcare, especially for Medicare Advantage plans. Cervical Cancer Screening (CCS) is a particularly high-impact one, but it’s also documentation-sensitive. That means, even if a patient did get the test, they might not get credit for it because it wasn’t coded correctly or wasn’t visible to the insurer.

So that’s the problem we set out to solve: How can we use the data that providers already have access to (or that we get through a third-party data provider like an HIE) and combine that with LLMs to validate whether CCS was met?

Let’s break down the measure quickly:
Women aged 21 to 30 are compliant if they’ve had a Pap smear in the last 2 years.
Women aged 30 to 64 need both a Pap smear and an HPV test in the last 4 years.
And we exclude anyone who’s had a hysterectomy.


Structured data for these tests might show up in various FHIR tables such as Procedure, DiagnosticReports, or Observations. But often the full story is hidden in HL7’s XML files: clinical notes, scanned documents, or free-text lab results.

**Why LLMs?**
There are two big reasons we bring in a large language model:
The structured data is ambiguous. For example, maybe the test was ordered but not completed, or the result says something vague like "N/A" or "see notes." That’s where we need the LLM to review contextual clues.
The test was done, but it’s not in the structured data at all. Maybe the result came from a different code system, or was buried in a narrative note. We use the LLM to extract that signal from the unstructured mess.

I created a basic SQL pipeline that queried all the relevant tables. For each patient, I generate a simple row with the following columns: Patient ID, Event Date, CPT Code, Test Result, Notes, and Link to the XML blob. 

**The Workflows**
From there, I proposed three main workflows:

Workflow 1: If the structured data is clean and clear, we apply Python logic only. We randomly sample 5% of these cases for human QA.

Workflow 2: If the data is ambiguous, we use the LLM to help us make a final call. We start by training the LLM on a stratified sample of known problem cases that have first been reviewed and scored by a human. After benchmarking and QA, we then switch to random sampling for final monitoring. 

Workflow 3: If there’s no structured signal at all, we pull the full XML record. After a bit of preprocessing, we hand it off to the LLM. And because these records can be long and messy, we require human review of at least 25% of the positive results.

**Key Considerations**
There are several key considerations when evaluating LLM performance in this context: sensitivity to inputs, types of outputs, and the human review process.

First, LLM results are sensitive to two main inputs: the underlying data and the prompt structure. In my initial testing, I found that changes to the included or excluded data had a much greater impact on results than variations in prompt wording. This suggests that careful data curation is more critical than prompt engineering for this use case. This was particularly true for Workflow 2 and why we went with stratified random sampling and a human-first process. We stratified the ambiguous cases into categories such as 1) Test ordered, no result, 2) Result present, but no date, 3) Evidence hidden in notes. From there We pulled even samples from each category and used them to train and evaluate the LLM.

Second, there are two distinct types of LLM outputs: the classification outcome (e.g., correctly identifying whether a patient should be included) and the justification (e.g., whether the LLM can point to supporting evidence and cite its source from the file). The reason that I suggested that we have clinicians review 1 in every 4 patient records in Workflow 3 is because some clinical notes are extremely long and complex, increasing the risk of hallucination or misinterpretation by the model.

Finally, the review process itself varied across workflows depending on the level of QA required. I proposed a human-first approach in Workflow 2, where benchmarking and validation were critical. In contrast, Workflows 1 and 3 relied on a human-reviewed model, where outputs were spot-checked after initial processing. All workflows included ongoing human review via random sampling to monitor for drift over time.

**My Takeaways**
Garbage in, garbage out: LLM performance depends more on input data than clever prompting.
Stratified sampling helps build a much more reliable dataset.
Human-in-the-loop QA is crucial, especially for long records where LLMs might hallucinate.



