<script setup lang="ts">
import { inject, onMounted, onUnmounted, ref } from "vue";
import _debounce from "lodash/debounce";

import { useHidePosterGuess } from "../../hooks/useHidePosterGuess";
import { useHideActivity } from "../../../../hooks/useHideActivity";
import { useCloseToStore } from "../../../../common/hooks/useCloseToStore";

import { useGuessPoster } from "./hooks/useGuessPoster";

import {
  KButton,
  KLoadingIcon,
  KLongNum,
  KScrollCountdown,
  SelectHeroAndSkin,
} from "@/components/business";
import { vMouseTip, vTypewriterSingle } from "@/directives";
import { _exitFullScreen, _openFullScreen } from "@/utils/tool";
import { $message } from "@/utils/busTransfer";
import { KnapsackStore } from "@/store";
import { GAME_PROP, MESSAGE_TIP } from "@/config";
import { _getPropLink } from "@/utils/concise";

const modelValue = defineModel<boolean>({ required: true });

const $knapsackStore = KnapsackStore();

const $useHidePosterGuess = useHidePosterGuess();
const $useHideActivity = useHideActivity();

/** 关闭活动 */
const closeActivity = inject<() => void>("close-activity")!;
const {
  answer_status,
  answer,
  countdown,
  guess_coin,
  guessing,
  loading,
  poster,
  randomPoster,
  receiveGuessCoin,
  reveal,
  right_answer,
  running,
  setShowStatus,
  show_receive,
  show_status,
  status_text,
  submitAnswer,
} = useGuessPoster(closeActivity, handleClose);

/** 是否显示 */
const show = ref(true);
/** 是否显示按钮 */
const show_btn = ref(false);

/* 打字结束后触发 */
const typewriterCallback = () => {
  show_btn.value = true;
};

/* 是否关闭游戏 */
function handleClose() {
  show.value = false;

  setTimeout(() => {
    modelValue.value = false;
    $useHidePosterGuess.setHideStatus(false);
    $useHideActivity.setHideStatus(false);
  }, 500);
}

/* 是否关闭游戏及活动 */
const closeAll = () => {
  handleClose();
  setTimeout(() => {
    closeActivity();
  }, 1500);
};

/* 是否进入下一题 */
const handleNext = () => {
  if (useCloseToStore(closeAll)) return;
  if (!show_btn.value) return;
  show_btn.value = false;
  randomPoster();

  setTimeout(() => {
    setShowStatus(false);
  }, 250);
};

/* 关闭竞猜 */
const debounceStopGuess = _debounce(() => {
  if (guessing.value) {
    $message(MESSAGE_TIP.z0r7, "error");
  } else {
    $message(MESSAGE_TIP.g3h9);

    //非正常退出自动领取奖励
    if (show_receive.value) {
      receiveGuessCoin();
    }
  }
  handleClose();
}, 100);

onMounted(() => {
  _openFullScreen();
  setTimeout(() => {
    if (window.innerWidth !== window.screen.width) {
      debounceStopGuess();
      return;
    }

    window.addEventListener("resize", debounceStopGuess);
    window.addEventListener("mouseleave", debounceStopGuess);
  }, 1000);
});

onUnmounted(() => {
  window.removeEventListener("resize", debounceStopGuess);
  window.removeEventListener("mouseleave", debounceStopGuess);
  _exitFullScreen();
});
</script>

<template>
  <teleport to="body">
    <transition name="fade" appear>
      <div v-show="show" class="guess-game">
        <!-- 剩余道具 -->
        <div class="guess-prop">
          <div class="prop">
            <img :src="_getPropLink(GAME_PROP.ICON['GUESS_CARD'])" alt="" class="icon" />
            <div class="num">{{ $knapsackStore.articles.GUESS_CARD }}</div>
          </div>
          <div class="prop">
            <img :src="_getPropLink(GAME_PROP.ICON['GUESS_COIN'])" alt="" class="icon" />
            <KLongNum class="k-long-num" :value="$knapsackStore.articles.GUESS_COIN" />
          </div>
        </div>

        <!-- 出题区 -->
        <div
          class="guess-poster"
          :class="{
            green: answer_status === 1,
            red: answer_status === 2,
          }"
        >
          <transition-group name="fade">
            <div v-show="show_status" key="c" class="answer">{{ right_answer }}</div>
            <KLoadingIcon v-if="loading" key="a" white width="6.25rem" />
            <div
              v-else
              key="b"
              class="topic"
              :style="{
                backgroundImage: `url(${poster})`,
              }"
              :class="{
                reveal,
              }"
            ></div>
          </transition-group>
        </div>

        <!-- 提交区域 -->
        <div class="submit">
          <SelectHeroAndSkin
            v-model="answer"
            :empty="false"
            :disabled="!running"
            black
            type="HERO"
            placeholder="英雄名称"
          />
          <div
            v-mouse-tip="{
              disabled: !running,
            }"
            class="commit"
            :class="{
              disabled: !running,
            }"
            @click="submitAnswer"
          >
            提交答案
          </div>
        </div>

        <!-- 领取奖励 -->
        <transition v-if="show_receive" name="fade" appear>
          <div class="receive-guess-coin">
            <div class="prop">
              <div
                class="icon"
                :style="{
                  backgroundImage: `url(${_getPropLink(GAME_PROP.ICON['GUESS_COIN'])})`,
                }"
              ></div>
              <div class="count">×{{ guess_coin }}</div>
            </div>
            <KButton type="warning" @click="receiveGuessCoin">领取竞猜币</KButton>
          </div>
        </transition>

        <!-- 回答状态 -->
        <transition v-else name="to-top">
          <div
            v-if="show_status"
            v-typewriter-single="{
              callback: typewriterCallback,
              delay: 250,
              speed: 100,
            }"
            class="answer-status"
            :class="{
              green: answer_status === 1,
              red: answer_status === 2,
            }"
            v-html="status_text"
          ></div>
        </transition>

        <!-- 是否进入下一题 -->
        <transition name="to-top">
          <div v-show="show_btn" class="btns">
            <div class="close" @click="handleClose">
              <i class="iconfont wzry-guanbi"></i>
            </div>
            <div class="next" @click="handleNext">
              <i
                class="iconfont wzry-dui"
                :class="answer_status === 1 ? 'wzry-you' : 'wzry-dui'"
              ></i>
            </div>
          </div>
        </transition>

        <!-- 倒计时 -->
        <transition name="to-top">
          <div v-show="running" class="countdown">
            <div class="text">剩余填写时间：</div>
            <transition name="fade">
              <KScrollCountdown :rate="0.45" :running="running" unit="SEC" :nums="countdown" />
            </transition>
          </div>
        </transition>

        <!-- Tip -->
        <transition name="to-top" appear>
          <div class="tip">电脑端只能在全屏下答题，一旦识别非全屏，将自动退出竞猜并扣除竞猜券</div>
        </transition>
      </div>
    </transition>
  </teleport>
</template>

<style scoped lang="less">
@import url("./index.less");
</style>