<script>
  import { createEventDispatcher } from "svelte";
  const dispatch = createEventDispatcher();
  import loading from "../../public/loading.gif";
  import ai from "../../public/icons_ai.svg";
  import Textarea from "../utils/Textarea.svelte";
  import bin from "../../public/icons_bin.svg";
  import { consequenceState } from "./store";

  export let projectData;
  export let consequenceSuggestions;
  export let consequences;
  export let onAdd;
  let AIPrompt;
  let getSuggest = false;

  let errorMessage = { message: null, status: false };
  let aiSuggest = false;
  let customConsequences;

  consequenceState.subscribe((value) => {
    customConsequences = value.unintendedIsComplete;
  });
  consequenceState.subscribe((value) => {
    AIPrompt = value.unintendedPrompt;
  });

  let fetchAttempts = 0;
  let isLoading = true;

  function retryFetch() {
    errorMessage.status = false;
    fetchAttempts = 0;
    suggestConsequences();
  }

  const HOST_NAME = import.meta.env.VITE_HOST_NAME;

  let HOST = HOST_NAME || "http://localhost:3000/";
  HOST += "openai-completion";

  function addOwnConsequences() {
    errorMessage.status = false;
    consequenceState.update((value) => {
      value.unintendedIsComplete = true;
      return value;
    });
    consequenceState.update((value) => {
      value.unintendedPrompt = false;
      return value;
    });
    aiSuggest = false;

    let selected = $consequenceSuggestions.filter((s) => s.isSelected);
    selected.forEach((s) => {
      consequences.push({
        description: s.description,
        outcome: s.selectedOutcome,
        impact: "",
        likelihood: "",
        action: { description: "", stakeholder: "", date: "" },
        AIM: "",
        KPI: "",
      });
    });
    $consequenceSuggestions = $consequenceSuggestions.map((a) => ({
      ...a,
      isSelected: false,
    }));
    consequenceSuggestions.set($consequenceSuggestions);
  }

  async function convertProjectDataToString() {
    const { objectives, title, stakeholders, dataUsed } = projectData;

    return `
        Project Title: ${title}
        Objectives: ${objectives}
        Stakeholders: ${stakeholders.map((s) => s.text)}
        Data Used: ${dataUsed}
        `;
  }

  async function reviewWithAI(content) {
    let promptContext = `
    Anticipate and address the potential impacts of a product or service on society. I need insights and suggestions based on the following project data:

${content}

Based on the information provided, please provide me with a list of 5 potential unintended consequences, including the description of the unintended consequence appropriate actions, impact, likelihood, AIM, timeline, KPI, outcome along with their impact (High/Medium/Low), outcome (Positive/Negative), likelihood (High/Medium/Low), action, measure (Act/Influence/Monitor), and timeline (3 months/6 months/1 year/2 years). Your output must an object of arrays in JSON format with the following structure:
   [
    {
    description : [Provide a Brief description of the unintended consequence],
    outcome: ["Negative" or "Positive"],
    impact: ["High" or "Medium" or "Low"],
    likelihood: ["High" or "Medium" or "Low"],
    action: {description: ["Suggested action"], stakeholder: ["Stakeholder responsible for completion of the action"], date : [A date in this format YYYY-MM-DD]},
    AIM: ['Act' or 'Influence' or 'Monitor'],
    KPI: [Suggested KPIs]
    isSelected: [true // always mark as true]
   },
   {
    description : [Provide a Brief description of the unintended consequence],
   outcome: ["Negative" or "Positive"],
    impact: ["High" or "Medium" or "Low"],
    likelihood: ["High" or "Medium" or "Low"],
    action: {description: ["Suggested action"], stakeholder: ["Stakeholder responsible for completion of the action"], date : [A date in this format YYYY-MM-DD]},
    AIM: ['Act' or 'Influence' or 'Monitor'],
    KPI: [Suggested KPIs]
    isSelected: [true // always mark as true]
   }
  ]
`;
    try {
      const review = await fetch(HOST, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          //  Authorization: 'Bearer ' + apiKey
        },
        body: JSON.stringify({
          messages: [{ role: "user", content: promptContext }],
          model: "gpt-3.5-turbo",
        }),
      });
      if (!review.ok) {
        throw new Error(
          `Network response was not ok, status: ${review.status}, statusText: ${review.statusText}`
        );
      }
      const data = await review.json();

      const suggestions = await data.choices[0].message.content;
      const suggestionsWithIsSelected = JSON.parse(suggestions).map(
        (suggestion) => {
          return {
            ...suggestion,
            description: suggestion.description
              .replace(/^\[\d+\]\.\s*/, "")
              .replace("Unintended consequence", "")
              .replace("Unintended Consequence", "")
              .replace(":", ""),
            isSelected: true,
          };
        }
      );
      return suggestionsWithIsSelected;
    } catch (error) {
      fetchAttempts++;
      console.error("Fetch attempt failed:", error);
      if (fetchAttempts >= 2) {
        errorMessage.message =
          "There was an error fetching the AI suggestions. You can add your own consequences.";
        errorMessage.status = true;
        return null;
      }
    }
  }

  async function suggestConsequences() {
    AIPrompt = false;
    getSuggest = true;
    aiSuggest = true;
    isLoading = true;
    const projectDataString = await convertProjectDataToString();
    let dataFromAI = await reviewWithAI(projectDataString);
    fetchAttempts++;
    if (!dataFromAI && fetchAttempts < 2) {
      console.log("No data returned from AI, retrying...");

      dataFromAI = await reviewWithAI(projectDataString);
      fetchAttempts++;
    }
    if (!dataFromAI && fetchAttempts >= 2) {
      console.log("No data returned from AI after two attempts.");
      errorMessage.message =
        "Failed to fetch AI suggestions. Please try again later or add your own consequences."; // Set an error message to inform the user.
      errorMessage.status = true;
    }
    if (dataFromAI) {
      isLoading = false;
      aiSuggest = true;
      consequenceSuggestions.update(() => dataFromAI);
      setTimeout(() => {
        window.scrollTo(0, document.body.scrollHeight);
      }, 200);
    } else {
      isLoading = false; // Ensure loading is set to false if no data is received after retries.
    }
  }

  function handleBinClick(selectedDescription) {
    const updatedSuggestions = $consequenceSuggestions.map((sug) => {
      if (sug.description === selectedDescription) {
        return {
          ...sug,
          isSelected: false,
        };
      }
      return sug;
    });
    consequenceSuggestions.set(updatedSuggestions);
  }
  async function onProceed() {
    consequenceState.update((currentState) => {
      currentState.unintendedIsComplete = true;
      return currentState;
    });
    consequenceState.update((currentState) => {
      currentState.unintendedPrompt = false;
      return currentState;
    });
    let selected = $consequenceSuggestions.filter((s) => s.isSelected);
    for (let i = consequences.length - 1; i >= 0; i--) {
      if (
        !consequences[i].description ||
        consequences[i].description.trim() === ""
      ) {
        consequences.splice(i, 1);
      }
    }
    selected.forEach((s) => {
      consequences.push({
        description: s.description,
        outcome: s.selectedOutcome,
        impact: "",
        likelihood: "",
        action: { description: "", stakeholder: "", date: "" },
        AIM: "",
        KPI: "",
      });
    });

    $consequenceSuggestions = $consequenceSuggestions.map((a) => ({
      ...a,
      isSelected: false,
    }));
    await consequenceSuggestions.set($consequenceSuggestions);
    dispatch("proceed", { details: projectData.unintendedConsequences });
  }
</script>

<div id="UnintendedConsequences">
  <div class="bg-orange-100 p-12">
    <div class="text-blue-800 mb-4 font-bold text-xl md:text-2xl">
      Unintended Consequences
    </div>
    <div class="mb-7 p-8 bg-white shadow-md">
      <div class="mb-4">
        To ensure a comprehensive risk assessment for your data project, you
        should try to identify any potential unintended consequences that may
        occur. These unintended consequences can have both positive and negative
        impacts. Consider the following:
      </div>
      <ul class="list-disc pl-5 mb-4">
        <li class="mb-2">
          <strong class="text-blue-500"
            >Positive Unintended Consequences:</strong
          > Think about unexpected benefits or positive outcomes that might result
          from your project. For example, improved processes, increased efficiency,
          or unforeseen uses or users of the product or service.
        </li>
        <li class="mb-2">
          <strong class="text-red-500">Negative Unintended Consequences:</strong
          > Consider any adverse effects or unexpected problems that might occur.
          This could involve privacy breaches, data misuse, or potential bias in
          the data.
        </li>
        <li class="mb-2">
          <strong>Secondary Effects:</strong> Think about ripple effects that your
          project might have on other processes, systems, or stakeholders, even if
          they are not directly involved.
        </li>
        <li class="mb-2">
          <strong>External Factors:</strong> Take into account external factors or
          events that your project might interact with or influence. These could
          include changes in regulations, market conditions, or technological advancements.
        </li>
      </ul>
      <div class="mb-2">
        Add a brief description of the unintended consequence that may occur
        during your data project and indicate whether the outcome is positive or
        negative.
      </div>
    </div>
  </div>
  {#if AIPrompt === true}
    <div class="bg-blue-100 p-12">
      <div class="flex">
        <img class="h-10 w-9 mr-5 filter-blue" src={ai} />
        <div class="text-blue-800 font-bold text-xl md:text-2xl">
          Do you want AI to suggest unintended consequences?
        </div>
      </div>
      <div class="mt-7 p-8 bg-white shadow-md bg-opacity-70">
        The AI will review the details of the project and make suggestions for
        what the possible intended and unintended consequences might be. The
        generated consequences should be treated as a guide that supports your
        project planning.
      </div>
      <div class="">
        <button
          class="my-5 bg-transparent text-blue-800 font-bold text-base border-blue-800 border-2 py-2 px-6"
          style="display"
          on:click={suggestConsequences}>Yes</button
        >
        <button
          class="my-5 bg-transparent text-blue-800 font-bold text-base border-blue-800 border-2 py-2 px-6"
          on:click={addOwnConsequences}>No</button
        >
      </div>
    </div>
  {/if}
  {#if getSuggest === true}
    <div class="bg-orange-100 p-12">
      <div class="mb-7 p-8 bg-white opacity-80 shadow-md">
        These consequences have been generated by the AI. Each generated
        consequence has also been identified as having either a positive or
        negative outcome. Review the consequences and edit or remove any that
        require amendments. You can also create your own consequences and add
        them to the list.
      </div>
      {#if isLoading}
        <div class="loading h-2">
          <span class="text-lg p-4">Loading...</span>
          <img alt="loading-icon ml-8 mt-1" src={loading} />
        </div>
      {/if}
      <div class="flex flex-col w-full justify-center">
        {#each $consequenceSuggestions as suggestion}
          <div class="relative">
            {#if suggestion.isSelected}
              <Textarea bind:value={suggestion.description} />
              <img
                src={bin}
                alt="bin icon"
                class="filter-blue absolute top-2 right-2 mt-1 pb-1 mr-4 cursor-pointer h-5"
                on:click={() => handleBinClick(suggestion.description)}
              />
            {/if}
          </div>
        {/each}
      </div>
    </div>
    {#if errorMessage.status}
      <div class="bg-orange-100 px-10 text-lg font-bold">
        {errorMessage.message}
        <div>
          <button
            class="my-5 bg-transparent text-blue-800 font-bold text-base border-blue-800 border-2 py-2 px-6"
            on:click={retryFetch}
          >
            Try Again
          </button>
        </div>
      </div>
    {/if}
    <div class="bg-orange-100 p-12">
      <button
        class="my-5 bg-transparent text-blue-800 font-bold text-base border-blue-800 border-2 py-2 px-6"
        on:click={onProceed}>Continue with these consequences</button
      >
      <button
        class="my-5 bg-transparent text-blue-800 font-bold text-base border-blue-800 border-2 py-2 px-6"
        on:click={addOwnConsequences}>Add in your own consequences</button
      >
    </div>
  {/if}
  {#if aiSuggest === false && customConsequences === true}
    <div id="UnintendedConsequences" class="bg-blue-100 p-12">
      <div class="text-blue-800 mb-4 font-bold text-xl md:text-3xl">
        Unintended Consequences
      </div>
      {#each consequences as consequence, i}
        <div class="consequence-options">
          <label for="description">
            <div class="text-blue-800 mb-4 font-bold text-lg md:text-md">
              Unintended Consequence {i + 1}
            </div>
            <Textarea
              bind:value={consequence.description}
              placeholder="Description"
            />
          </label>
          <div class="input-row">
            <label for="outcome">
              <span class="text-blue-800 font-bold text-lg">Outcome</span>
              <select bind:value={consequence.selectedOutcome}>
                <option disabled value>Outcome</option>
                <option
                  selected={consequence.selectedOutcome === "Positive"}
                  value="Positive">Positive</option
                >
                <option
                  selected={consequence.selectedOutcome === "Negative"}
                  value="Negative">Negative</option
                >
              </select>
            </label>
          </div>
        </div>
      {/each}
      <div>
        <button
          class="m-5 bg-transparent text-blue-800 font-bold text-base border-blue-800 border-2 py-2 px-3"
          on:click={onAdd}>Add More</button
        >
        <button
          class="m-5 bg-transparent text-blue-800 font-bold text-base border-blue-800 border-2 py-2 px-3"
          on:click={onProceed}>Proceed to Evaluate Risk</button
        >
      </div>
    </div>
  {/if}
</div>
