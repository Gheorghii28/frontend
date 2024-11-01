<template>
  <div>
    <InputBtn class="my-7" full @click="openDialogAndAddNew()">
      <PlusCircleIcon class="h-5 w-5" />
      {{ t("Buttons.AddNew", { item: t("Buttons.Quiz") }) }}
    </InputBtn>
    <div 
      v-if="quizzes?.length"
      class="pb-4 pr-4 pl-4 xl:pb-5 xl:pr-5 xl:pl-5">
      <article class="flex justify-between">
        <div>
          <p class="text-sm text-accent clamp line-1 tight">
            {{ t("Headings.Total") }}: {{ filteredQuizzes?.length }}
            {{ filteredQuizzes.length === 1 ? t("Headings.Quiz") : t("Headings.Quizzes") }}
          </p>
          <p class="text-sm clamp line-1 tight">
            {{ t("Headings.QuizCreated", { 
              count: userQuizCount, 
              quizText: userQuizCount === 1 ? t("Headings.Quiz") : t("Headings.Quizzes") 
            }) }}
          </p>
        </div>
        <div class="flex gap-4 mb-4">
          <InputSelect
            id="select_user"
			    	sm
			    	:options="formattedUsers"
			    	btn-type
			    	v-model="selectedUser"
            @change="filterQuizzes"
			    />
          <InputSelect
            id="select_quiz_type"
			    	sm
			    	:options="availableTypes"
			    	btn-type
			    	v-model="selectedType"
            @change="filterQuizzes"
			    />
        </div>
      </article>
    </div>
    <section v-if="filteredQuizzes?.length">
      <div
        v-for="(quiz, i) of filteredQuizzes"
        :key="i"
        class="p-4 xl:p-5 bg-secondary mb-4 rounded-md"
      >
        <article class="flex justify-between gap-4">
          <p class="clamp line-1 tight sm:pr-3 md:pr-5 md:w-4/5 lg:w-2/3">
            {{ quiz?.question ?? "" }}
          </p>

          <div class="flex gap-3 items-center">
            <TrashIcon
              v-if="!!user?.admin || quiz?.creator === user?.id"
              @click="fnDeleteQuiz(quiz?.id)"
              class="h-5 w-5 cursor-pointer text-accent"
            />
            <PencilSquareIcon
              v-if="!!user?.admin || quiz?.creator === user?.id"
              @click="openDialogCreate(quiz)"
              class="h-5 w-5 cursor-pointer text-accent"
            />
            <EyeIcon
              v-else-if="!user?.admin"
              @click="openDialogCreate(quiz)"
              class="h-5 w-5 cursor-pointer text-accent"
            />
          </div>
        </article>
        <div class="flex flex-col sm:flex-row items-start gap-1">
          <p class="text-xs text-accent clamp line-1 tight sm:pr-3 md:pr-53">
            {{ quiz?.creator === user?.id ? t("Headings.QuizCreatedByYou") : t("Body.CreatedBy", { user: quiz?.creator})  }}
          </p>
          <p class="text-xs text-subheading clamp line-1 tight sm:pr-3 md:pr-5">
            {{ quiz?.type  }}
          </p>
        </div>
      </div>
    </section>
    <p
      class="border border-accent rounded-md w-full p-5 text-xl text-center"
      v-else
    >
      {{ t("Headings.EmptyQuizzes") }}
    </p>
    <div>
      <DialogSlot
        v-if="dialogCreateQiuz"
        :label="'Headings.Quiz'"
        :propClass="'modal-width-lg lg:modal-width-md'"
        :show="dialog"
        @closeFunction="dialogCreateQiuz = false"
      >
        <LazyFormQuiz :data="propData" :taskId="taskId" />
      </DialogSlot>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { useDialogSlot } from "@/composables/dialogSlot";
import { useI18n } from "vue-i18n";
import {
  TrashIcon,
  EyeIcon,
  PencilSquareIcon,
  PlusCircleIcon,
} from "@heroicons/vue/24/outline";
import type { Quiz } from "~/types/courseTypes";

const props = defineProps({
  quizzes: { type: Array as PropType<any>, default: [] },
  taskId: { type: String, default: "" },
});
const { t } = useI18n();
const dialog = useDialogSlot();
const dialogCreateQiuz = useDialogCreateSubtask();
const selectedUser = ref();
const selectedType = ref();
const quizzes = ref(props.quizzes);
const filteredQuizzes = ref([...quizzes.value]);
const propData = ref();
const user: any = useUser();

const userQuizCount = computed(() => filteredQuizzes.value.filter((quiz) => quiz.creator === user.id).length);

const formattedUsers = computed<{ label: string; value: string }[]>(() => {
  const creatorsSet = new Set<string>(quizzes.value.map((quiz: Quiz) => quiz.creator));
  const creatorOptions = Array.from(creatorsSet).map(creatorId => {
    const creatorDisplayName = quizzes.value.find((quiz: Quiz) => quiz.creator === creatorId)?.creator || "Unknown";
    return {
      label: `${creatorDisplayName}${creatorId === user.id ? ' (You)' : ''}`,
      value: creatorId,
    };
  });

  return [{ label: t("List.Filter.AllUsers"), value: "" }, ...creatorOptions];
});

const availableTypes = computed<{ label: string; value: string }[]>(() => {
  const typesSet = new Set(quizzes.value.map((quiz: Quiz) => quiz.type));
  const typeOptions = Array.from(typesSet).map(type => ({ label: String(type), value: String(type) }));

  return [{ label: t("List.Filter.AllTypes"), value: "" }, ...typeOptions];
});

onMounted(() => {
  selectedUser.value = "";
  selectedType.value = "";
});

function filterQuizzes() {
  filteredQuizzes.value = quizzes.value.filter((quiz: Quiz) => {
    const userMatches = selectedUser.value ? quiz.creator === selectedUser.value : true;
    const typeMatches = selectedType.value ? quiz.type === selectedType.value : true;

    return userMatches && typeMatches;
  });
}

function openDialogCreate(quiz: any) {
  propData.value = quiz;
  dialog.value = true;
  dialogCreateQiuz.value = true;
}

function openDialogAndAddNew() {
  propData.value = null;
  dialog.value = true;
  dialogCreateQiuz.value = true;
}
function fnDeleteQuiz(id: any) {
  openDialog(
    "info",
    "Headings.DeleteQuiz",
    "Body.DeleteQuiz",
    false,
    {
      label: "Buttons.DeleteQuiz",
      onclick: async () => {
        const [success, error] = await deleteSubTaskInQuiz(props.taskId, id);

        if (success) {
          setTimeout(() => {
            openSnackbar("success", "Success.DeleteQuiz");
          }, 1000);
        } else {
          openSnackbar("error", error?.detail ?? "");
        }
      },
    },
    {
      label: "Buttons.Cancel",
      onclick: () => {},
    }
  );
}
</script>

<style scoped></style>
