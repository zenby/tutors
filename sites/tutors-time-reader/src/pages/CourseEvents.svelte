<script lang="ts">
  import CardDeck from "../components/cards/CardDeck.svelte";
  import type { Lo } from "tutors-reader-lib/src/types/lo-types";
  import { layout, portfolio } from "../stores";
  import { fetchAllCourseAccess } from "tutors-reader-lib/src/utils/firebase-utils";
  import { child, get, getDatabase, onValue, ref } from "firebase/database";
  import { toHoursAndMinutes } from "tutors-reader-lib/src/utils/metrics-utils";

  let los: Lo[] = [];
  let canUpdate = false;
  let courseMap = new Map<string, any>();
  $: numberModules = 0;
  let title = "Tutors Time";
  let subTitle = "Connecting....";

  function summarise(usage: any, visits: number): string {
    let str = "";
    if (usage.count) str += `${toHoursAndMinutes(usage.count)}<br>`;
    str += `${usage.visits}<br>`;
    str += `${visits}<br>`;
    str += "<hr><br>";
    str += `${usage.title}`;
    return str;
  }

  layout.set("compacted");
  async function getAllCourses(): Promise<Lo[]> {
    portfolio.set(true);
    const db = getDatabase();
    let allCourseAccess = await fetchAllCourseAccess();
    const func = () => {
      canUpdate = true;
      subTitle = "";
    };
    setTimeout(func, 20 * 1000);
    for (let i = 0; i < allCourseAccess.length; i++) {
      const courseId = allCourseAccess[i].courseId;
      const statusRef = ref(db, `all-course-access/${courseId}/visits`);
      onValue(statusRef, async () => {
        if (canUpdate) {
          const snapshot = await get(child(ref(db), `all-course-access/${courseId}`));
          const usage = snapshot.val();
          if (usage.lo) {
            const lo = usage.lo;
            const foundLo = courseMap.get(courseId);
            if (foundLo) {
              foundLo.title = lo.courseTitle;
              foundLo.visits++;
              foundLo.summary = summarise(usage, foundLo.visits);
              foundLo.route = `https://reader.tutors.dev${lo.subRoute}`;
              if (lo.img) {
                foundLo.img = lo.img;
                if (lo.icon) {
                  foundLo.icon = lo.icon;
                } else {
                  foundLo.icon = null;
                }
              }
            } else {
              if (!lo.isPrivate) {
                const loCopy = { ...lo };
                loCopy.title = lo.courseTitle;
                loCopy.route = `https://reader.tutors.dev${lo.subRoute}`;
                loCopy.visits = 1;
                loCopy.summary = summarise(usage, 1);
                loCopy.type = "web";
                courseMap.set(courseId, loCopy);
                los.push(loCopy);
              }
            }
            // los.sort((a: any, b: any) => b?.visits - a?.visits);
            los = [...los];
            numberModules = los.length;
          }
        }
      });
    }
    return los;
  }
</script>

<svelte:head>
  <title>{title}</title>
</svelte:head>

<div class="header-container navbar">
  <div class="flex-1">
    <div class="navbar-title">
      <p class="text-center text-lg">
        Tutors Time: {numberModules} Modules Currently Online : {new Date().toLocaleString()}
        {subTitle}
      </p>
    </div>
  </div>
</div>
<div class="container mx-auto">
  {#await getAllCourses()}
    <h1>waiting</h1>
  {:then courses}
    <CardDeck {los} />
  {/await}
</div>
