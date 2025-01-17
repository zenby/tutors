<script lang="ts">
  import { getContext, onDestroy } from "svelte";
  import { fade } from "svelte/transition";
  import LabTime from "./support/LabTime.svelte";
  import InstructorLabTime from "./support/InstructorLabTime.svelte";
  import CalendarTime from "./support/CalendarTime.svelte";
  import InstructorCalendarTime from "./support/InstructorCalendarTime.svelte";
  import type { Course } from "tutors-reader-lib/src/models/course";
  import type { CourseService } from "../reader-lib/services/course-service";
  import { currentLo, currentUser } from "../stores";
  import { Tab, TabList, TabPanel, Tabs } from "svelte-tabs";
  import { querystring } from "svelte-spa-router";
  import type { MetricsService } from "src/reader-lib/services/metrics-service";

  export let params: Record<string, string> = {};

  let instructorMode = false;
  let course: Course;
  const cache: CourseService = getContext("cache");
  const metricsService: MetricsService = getContext("metrics");
  let title = "";
  let pinBuffer = "";
  let ignorePin = "";
  let id = "";

  window.addEventListener("keydown", keypressInput);

  async function getCourse(url: string) {
    id = $querystring;

    course = await cache.readCourse(url);
    metricsService.setCourse(course);
    const user = await metricsService.fetchUserById(id);
    currentUser.set(user);
    // noinspection TypeScriptValidateTypes
    currentLo.set({ title: `Tutors Time`, type: "tutorsTime", parentLo: course.lo, img: course.lo.img });
    title = `Tutors Time`;
    if (course.lo.properties.ignorepin) {
      ignorePin = "" + course.lo.properties.ignorepin;
    }
    return course;
  }

  function keypressInput(e: KeyboardEvent) {
    pinBuffer = pinBuffer.concat(e.key);
    if (pinBuffer === ignorePin) {
      instructorMode = true;
    }
  }

  onDestroy(() => {
    window.removeEventListener("keydown", keypressInput);
  });
</script>

<svelte:head>
  <title>{title}</title>
  <link rel="stylesheet" href="https://unpkg.com/ag-grid-community/dist/styles/ag-grid.css" />
  <link rel="stylesheet" href="https://unpkg.com/ag-grid-community/dist/styles/ag-theme-balham.css" />
</svelte:head>

{#await getCourse(params.wild) then course}
  <div in:fade={{ duration: 500 }} class="bg-base-200 mt-4 container mx-auto rounded-box">
    <Tabs>
      <TabList>
        <Tab>Labs</Tab>
        <Tab>Calendar</Tab>
        {#if instructorMode}
          <Tab>Labs All Students</Tab>
          <Tab>Calendar All Students</Tab>
        {/if}
      </TabList>

      <TabPanel>
        <LabTime {id} />
      </TabPanel>
      <TabPanel>
        <CalendarTime {id} />
      </TabPanel>
      {#if instructorMode}
        <TabPanel>
          <InstructorLabTime />
        </TabPanel>
        <TabPanel>
          <InstructorCalendarTime />
        </TabPanel>
      {/if}
    </Tabs>
  </div>
{/await}
