<template>
  <div
    class="note"
    :class="{
      selected: noteState === 'SELECTED',
      overlapping: noteState === 'OVERLAPPING',
      invalid_lyric_data: !isValidLyricData,
      invalid_lyric_input: !isValidLyricInput,
    }"
    :style="{
      width: `${width}px`,
      height: `${height}px`,
      transform: `translate3d(${positionX}px,${positionY}px,0)`,
    }"
  >
    <div class="note-bar" @mousedown="onBarMouseDown">
      <div class="note-left-edge" @mousedown="onLeftEdgeMouseDown"></div>
      <div class="note-right-edge" @mousedown="onRightEdgeMouseDown"></div>
      <context-menu ref="contextMenu" :menudata="contextMenuData" />
    </div>
    <div class="note-lyric" @mousedown="onLyricMouseDown">
      {{ lyric }}
    </div>
    <input
      v-if="showLyricInput"
      v-model.lazy.trim="lyric"
      v-focus
      class="note-lyric-input"
      :class="{ invalid: !isValidLyricInput }"
      @mousedown.stop
      @dblclick.stop
      @keydown.stop="onLyricInputKeyDown"
      @blur="onLyricInputBlur"
    />
    <q-tooltip
      v-if="showInvalidLyricDataTooltip && !showLyricInput"
      :offset="[8, 4]"
      anchor="bottom left"
      self="top left"
    >
      ひらがな/カタカナ1文字(データエラー)
    </q-tooltip>
    <q-tooltip
      v-model="showInvalidLyricInputTooltip"
      :offset="[8, 4]"
      anchor="bottom left"
      self="top left"
    >
      ひらがな/カタカナ1文字
    </q-tooltip>
  </div>
</template>

<script setup lang="ts">
import hasSmallLetter from "jaco/fn/hasSmallLetter";
import isOnlyHiragana from "jaco/fn/isOnlyHiragana";
import isOnlyKatakana from "jaco/fn/isOnlyKatakana";
import { computed, ref } from "vue";
import { useStore } from "@/store";
import { Note } from "@/store/type";
import {
  getKeyBaseHeight,
  tickToBaseX,
  noteNumberToBaseY,
} from "@/sing/viewHelper";
import ContextMenu from "@/components/Menu/ContextMenu.vue";
import { MenuItemButton } from "@/components/Menu/type";

type NoteState = "NORMAL" | "SELECTED" | "OVERLAPPING";

const vFocus = {
  mounted(el: HTMLInputElement) {
    el.focus();
    el.select();
  },
};

const props = withDefaults(
  defineProps<{
    note: Note;
    isSelected: boolean;
  }>(),
  {
    isSelected: false,
  }
);

const emit =
  defineEmits<{
    (name: "barMousedown", event: MouseEvent): void;
    (name: "rightEdgeMousedown", event: MouseEvent): void;
    (name: "leftEdgeMousedown", event: MouseEvent): void;
    (name: "lyricMouseDown", event: MouseEvent): void;
  }>();

const store = useStore();
const state = store.state;
const tpqn = computed(() => state.tpqn);
const zoomX = computed(() => state.sequencerZoomX);
const zoomY = computed(() => state.sequencerZoomY);
const positionX = computed(() => {
  const noteStartTicks = props.note.position;
  return tickToBaseX(noteStartTicks, tpqn.value) * zoomX.value;
});
const positionY = computed(() => {
  const noteNumber = props.note.noteNumber;
  return noteNumberToBaseY(noteNumber + 0.5) * zoomY.value;
});
const height = computed(() => getKeyBaseHeight() * zoomY.value);
const width = computed(() => {
  const noteStartTicks = props.note.position;
  const noteEndTicks = props.note.position + props.note.duration;
  const noteStartBaseX = tickToBaseX(noteStartTicks, tpqn.value);
  const noteEndBaseX = tickToBaseX(noteEndTicks, tpqn.value);
  return (noteEndBaseX - noteStartBaseX) * zoomX.value;
});
const noteState = computed((): NoteState => {
  if (props.isSelected) {
    return "SELECTED";
  }
  if (state.overlappingNoteIds.has(props.note.id)) {
    return "OVERLAPPING";
  }
  return "NORMAL";
});
const lyric = computed({
  get() {
    return props.note.lyric;
  },
  set(value) {
    const note: Note = { ...props.note, lyric: value };
    store.dispatch("UPDATE_NOTES", { notes: [note] });
  },
});
const showLyricInput = computed(() => {
  return state.editingLyricNoteId === props.note.id;
});

// NOTE: バリデーションおよび動作メモ
// 入力時のリアルタイムバリデーションとデータのバリデーションの両方を行いたい
// 入力時: 空または許容する1モーラ以外の場合かつIME変換中でない場合にエラー・ちらついて邪魔なのでアニメーションしないようにする
// データ: データのバリデーションはAPIからレスポンスを取得した後に行う(返却に含まれる)
//
// たぶんすでにどこかにバリデーション実装はありそう…
// input要素とノートのバーはcomponentとしてわけたほうがよさそう
// リアルタイムバリデーションは実行タイミングを修正すべき…

// 入力中の歌詞テキストがバリデーションエラーかどうか(onkeydown時に更新)
const isValidLyricInput = ref(false);

// インポートなどの歌詞テキストがバリデーションエラーかどうか
const isValidLyricData = computed(() => isValidLyric(lyric.value));

const showInvalidLyricDataTooltip = computed(() => {
  return !isValidLyricData.value && !showLyricInput.value;
});

// 入力時バリデーションエラー時にツールチップを表示するかどうか
const showInvalidLyricInputTooltip = computed(() => {
  return !isValidLyricInput.value && showLyricInput.value;
});

const contextMenu = ref<InstanceType<typeof ContextMenu>>();
const contextMenuData = ref<[MenuItemButton]>([
  {
    type: "button",
    label: "削除",
    onClick: async () => {
      contextMenu.value?.hide();
      store.dispatch("REMOVE_SELECTED_NOTES");
    },
    disableWhenUiLocked: true,
  },
]);

const onBarMouseDown = (event: MouseEvent) => {
  emit("barMousedown", event);
};

const onRightEdgeMouseDown = (event: MouseEvent) => {
  emit("rightEdgeMousedown", event);
};

const onLeftEdgeMouseDown = (event: MouseEvent) => {
  emit("leftEdgeMousedown", event);
};

const onLyricMouseDown = (event: MouseEvent) => {
  emit("lyricMouseDown", event);
};

const onLyricInputKeyDown = (event: KeyboardEvent) => {
  // タブキーで次のノート入力に移動
  if (event.key === "Tab") {
    event.preventDefault();
    const noteId = props.note.id;
    const notes = store.getters.SELECTED_TRACK.notes;
    const index = notes.findIndex((value) => value.id === noteId);
    if (index === -1) {
      return;
    }
    if (event.shiftKey && index - 1 < 0) {
      return;
    }
    if (!event.shiftKey && index + 1 >= notes.length) {
      return;
    }
    const nextNoteId = notes[index + (event.shiftKey ? -1 : 1)].id;
    store.dispatch("SET_EDITING_LYRIC_NOTE_ID", { noteId: nextNoteId });
  }
  // 入力値をバリデーションする
  const inputElement = event.target as HTMLInputElement;
  if (isValidLyric(inputElement.value) || !event.isComposing) {
    isValidLyricInput.value = true;
  } else {
    isValidLyricInput.value = false;
  }
  // IME変換確定時のEnterを無視する
  if (event.key === "Enter" && event.isComposing) {
    return;
  }
  // IME変換でなければ入力モードを終了
  if (event.key === "Enter" && !event.isComposing) {
    store.dispatch("SET_EDITING_LYRIC_NOTE_ID", { noteId: undefined });
  }
};

const onLyricInputBlur = () => {
  if (state.editingLyricNoteId === props.note.id) {
    store.dispatch("SET_EDITING_LYRIC_NOTE_ID", { noteId: undefined });
  }
};

// 日本語1モーラまでのバリデーション
// NOTE: データのバリデーション(レスポンスから取得する)と入力中のリアルタイムバリデーションを分けるべき
// また、即時バリデーションを行うなら、エンジンが対応するモーラ辞書と対比させるべき
const isValidLyric = (lyricText: string) => {
  // ほんとは空白文字も許容しない(事前trimでもいいかも)
  // 空文字は許容しない
  if (lyricText === "") {
    return false;
  }
  // ひらがな・カタカナ以外は許容しない
  if (!isOnlyHiragana(lyricText) || isOnlyKatakana(lyricText)) {
    return false;
  }
  // 2文字以上の場合、小文字が含まれていなければ許容しない(ゔぁなどは許可)
  if (lyricText.length > 1 && !hasSmallLetter(lyricText)) {
    return false;
  }

  return true;
};
</script>

<style scoped lang="scss">
@use '@/styles/variables' as vars;
@use '@/styles/colors' as colors;

.note {
  position: absolute;
  top: 0;
  left: 0;

  &.selected {
    // 色は仮
    .note-bar {
      background-color: hsl(33, 100%, 50%);
    }
  }

  &.overlapping {
    .note-bar {
      background-color: hsl(130, 35%, 85%);
    }
  }

  &.invalid_lyric_data {
    .note-bar {
      background: colors.$warning;
    }

    .note-lyric-input {
      border-color: colors.$warning;
    }
  }

  .invalid_lyric_input {
    .note-lyric-input {
      border-color: colors.$warning;
    }
  }
}

.note-lyric {
  box-sizing: border-box;
  position: absolute;
  left: 0.125rem;
  bottom: 0;
  padding: 0;
  background: transparent;
  color: #121212;
  font-size: 1rem;
  font-weight: 700;
  text-shadow: -1px -1px 0 #fff, 1px -1px 0 #fff, -1px 1px 0 #fff,
    1px 1px 0 #fff;
  white-space: nowrap;
  pointer-events: none;

  &::after {
    content: "";
    position: absolute;
    bottom: 0rem;
    left: 0;
    width: 100%;
    height: 0.25rem;
    background-color: rgba(0, 0, 0, 0.3); // NOTE: カラー決定後に変更
  }
}

.note-bar {
  position: absolute;
  width: calc(100% + 1px);
  height: 100%;
  background-color: colors.$primary;
  border: 1px solid rgba(colors.$background-rgb, 0.5);
  border-radius: 2px;
  cursor: move;
}

.note-left-edge {
  position: absolute;
  top: 0;
  left: -1px;
  width: 5px;
  height: 100%;
  cursor: ew-resize;
}

.note-right-edge {
  position: absolute;
  top: 0;
  right: -1px;
  width: 5px;
  height: 100%;
  cursor: ew-resize;
}

.note-lyric-input {
  position: absolute;
  bottom: 0;
  font-weight: 700;
  min-width: 2rem;
  max-width: 4rem;
  border: 1px solid hsl(33, 100%, 73%);
  border-radius: 0.25rem;

  &.invalid {
    border-color: colors.$warning;
  }
}
</style>
