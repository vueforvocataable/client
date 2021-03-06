<template>
  <div>
    <b-container id="text-field-page-container" class="px-0">
      <!-- progress bar -->
      <b-row v-if="showProgressGircular">
        <v-progress-circular
          class="mx-auto"
          style="filter: invert(26%) sepia(36%) saturate(2827%) hue-rotate(324deg) brightness(101%) contrast(93%); width:8rem; height:8rem;"
          indeterminate
        ></v-progress-circular>
      </b-row>

      <!-- Stepper -->
      <b-row v-if="stepper.isFirstVisit" id="stepper-container" class="mx-auto mb-5">
        <stepper></stepper>
      </b-row>

      <!-- 버튼 그룹 -->
      <b-row class="mx-auto">
        <b-button-group size="sm">
          <b-button v-on:click="downloadVoca()">
            <b-img :src="images.memo" alt="btn image" class="img-color"/>
            <span class="btn-font-size">메모장으로 저장</span>
          </b-button>
          <b-button
            :state="!validateText.validation"
            :disabled="!validateText.validation"
            @click="sendVocaToTable()"
          >
            <b-img :src="validateText.img" alt="btn image" class="img-color"/>
            <span class="btn-font-size">단어시험지 만들기</span>
          </b-button>
        </b-button-group>
      </b-row>

      <!-- 텍스트에어리아 -->
      <b-row class="mt-2" id="t-p-c-container">
        <b-col class="col1" cols="2">
          <b-list-group>
            <b-list-group-item disabled>카테고리</b-list-group-item>
            <b-list-group-item
              button
              @click="buttonGroup.category.text='TOEIC'; activeCategoryButton(0)"
              :active="buttonGroup.category.active[0]"
            >TOEIC</b-list-group-item>
            <b-list-group-item
              button
              @click="buttonGroup.category.text='TEPS'; activeCategoryButton(1)"
              :active="buttonGroup.category.active[1]"
            >TEPS</b-list-group-item>
            <b-list-group-item disabled>일본어</b-list-group-item>
            <b-list-group-item
              button
              @click="buttonGroup.category.text='JPLT'; activeCategoryButton(2)"
              :active="buttonGroup.category.active[2]"
            >JPLT</b-list-group-item>
            <b-list-group-item
              button
              @click="buttonGroup.category.text='ETC'; activeCategoryButton(3)"
              :active="buttonGroup.category.active[3]"
            >기타</b-list-group-item>
          </b-list-group>
        </b-col>

        <b-col class="col2" cols="4">
          <b-form-textarea
            placeholder="영어, 한글
  Simple, 간단한
  Voca, 단어
  Test paper, 시험지 "
            title="사용법"
            autofocus
            class="textfield"
            id="inputField"
            no-resize
            v-model="text"
          />
        </b-col>

        <!-- 프리뷰 -->
        <b-col cols="6" class="col3 preview-container">
          <div id="preview-label">미리보기</div>
          <preview id="preview" :vocaProp="voca" :vocaHeaderProp="vocaHeader"></preview>
        </b-col>
      </b-row>

      <!-- 텍스트에어리아 호버 툴팁 -->
      <b-tooltip target="inputField" placement="right">
        <span>
          각 단어 사이는
          <strong>,</strong> 로 구분합니다.
        </span>
      </b-tooltip>

      <!-- 다른 사용자가 사용한 단어를 카테고리로 정렬 후 불러옴 -->
      <category class="mt-5" v-on:copyText="copyText"></category>

      <!-- 사용자가 사용한 단어를 불러옴 -->
      <div id="my-words-container">
        <div class="mt-5" id="my-words-label">
          <span>내가 사용한 단어</span>
        </div>
        <my-words
          class="mt-1"
          :vocaProp="voca"
          :vocaHeaderProp="vocaHeader"
          :myWords="myWords"
          v-on:copyText="copyText"
        ></my-words>
      </div>
    </b-container>

    <!-- 스낵바 -->
    <snackbar></snackbar>
  </div>
</template>

<script>
import Preview from "./Preview.vue";
import category from "./category.vue";
import Snackbar from "./Snackbar.vue";
import MyWords from "./MyWords.vue";
import Stepper from "./Stepper.vue";
import { saveAs } from "@elastic/filesaver";
import axios from "axios";

export default {
  name: "VocaTextField",
  components: {
    "my-words": MyWords,
    preview: Preview,
    category: category,
    snackbar: Snackbar,
    stepper: Stepper
  },
  data() {
    return {
      showProgressGircular: false,
      //텍스트 에이리어에 있는 텍스트를 담는 변수
      text: "",
      //텍스트를 리폼한 단어를 담는 변수
      voca: [],
      vocaHeader: [],
      serverUrl: "https://vocatestsserver.herokuapp.com",
      // serverUrl: "http://localhost:5001",
      remoteVocas: [],
      images: {
        check: require("../../assets/check.svg"),
        uncheck: require("../../assets/uncheck.svg"),
        memo: require("../../assets/memo.svg"),
        arrow: require("../../assets/arrow.svg")
      },
      buttonGroup: {
        category: {
          text: "ETC",
          active: [false, false, false, true]
        }
      },
      myWords: [],
      stepper: {
        isFirstVisit: false
      }
    };
  },
  watch: {
    text: function(text) {
      let reformedText = this.reformText(text);
      this.voca = this.formatTextToVoca(reformedText);
    }
  },
  computed: {
    validateText: function() {
      const csvRegexp = /^[^,]+(,[^,]*)$/; //단어, 단어 이런 형식인지 판별
      if (this.text.length == 0) {
        return {
          validation: false,
          img: this.images.uncheck
        };
      }
      const vocas = this.text.split("\n");

      for (let i = 0; i < vocas.length; i++) {
        if (vocas[i] == "") continue; //공백일 경우 스킵

        const result = csvRegexp.test(vocas[i]);

        //정규식을 통과하지 못 하면 사진과 false반환
        if (result == false) {
          return {
            validation: false,
            img: this.images.uncheck
          };
        }
      }
      return {
        validation: true,
        img: this.images.check
      };
    }
  },
  created() {
    this.checkVisited();
  },
  methods: {
    checkVisited: function() {
      let isFirstVisit = false;
      try {
        isFirstVisit = localStorage.getItem("isFirstVisit");
      } catch (err) {
        console.log(err);
        isFirstVisit = true;
      }

      if (isFirstVisit == null || isFirstVisit == true) {
        //방문했음으로 방문처리
        this.stepper.isFirstVisit = true;
        localStorage.setItem("isFirstVisit", false);
      } else {
        this.stepper.isFirstVisit = false;
      }
    },
    activeCategoryButton: function(index) {
      for (let i = 0; i < this.buttonGroup.category.active.length; i++) {
        this.buttonGroup.category.active[i] = false;
      }
      this.buttonGroup.category.active.splice(index, 1, true);
    },
    copyText: function(text) {
      this.text = text;
    },
    sendVocaAsync: async function(router, text) {
      try {
        let query = {
          voca: text,
          category: this.buttonGroup.category.text
        }
        let res = await axios.post(this.serverUrl + router, query);

        const myWords = {
          voca: this.voca,
          vocaHeader: this.vocaHeader,
          category: res.data.savedVoca.category ? res.data.savedVoca.category : undefined,
          date: res.data.savedVoca.date ? res.data.savedVoca.date : undefined,
          id: res.data.savedVoca._id ? res.data.savedVoca._id : undefined
        };

        return myWords
      } catch (err) {
        console.log(err);
      }
    },
    postVocas: function(voca) {
      if (!voca || voca.length < 1) return;
      let router = "/api/voca";

      let text = "";
      text += `${this.vocaHeader[0].english}, ${this.vocaHeader[0].korean}\n`;
      voca.forEach((x, index) => {
        text += `${voca[index].english}, ${voca[index].korean}\n`;
      });

      this.showProgressGircular = true;

      return this.sendVocaAsync(router, text);
    },
    //입력받은 텍스트를 다듬은 후, 문자열 배열로 바꿔줌
    reformText: function(text) {
      text = text
        .replace(/\n/g, ",")
        .split(",") //엔터값없애줌
        .map(item => {
          //공백없애줌
          return item.trim();
        })
        .filter(item => {
          //""값 없애줌
          return item != "";
        });

      return text;
    },
    //유저가 작성한 텍스트를 테이블에 보낼 형식으로 바꿔주는 함수
    formatTextToVoca: function(text) {
      let vocaObj = new Array();
      let cnt = 0;
      let englishItemTemp;

      text.forEach(item => {
        cnt++;
        if (cnt % 2 == 1) {
          englishItemTemp = item;
        } else {
          vocaObj.push({
            english: englishItemTemp,
            korean: item
          });
        }
      });
      this.vocaHeader = vocaObj.splice(0, 1);

      return vocaObj;
    },
    //버튼클릭시 App.vue로 값을 보냄
    sendVocaToTable: async function() {
      let reformedText = this.reformText(this.text);
      this.voca = this.formatTextToVoca(reformedText);

      this.myWords.push(await this.postVocas(this.voca))

      //router에서 table로 값을 전달함
      this.$router.push({
        name: "Table",
        params: {
          vocaProp: this.voca,
          tableHeaderProp: this.vocaHeader
        }
      });
    },
    //파일 저장 버튼 누르면 실행
    //텍스트 필드의 텍스트를 파일로 저장함
    downloadVoca: function() {
      if (this.text === undefined || this.text.length < 1) return;

      const text = this.text;
      const newlineRegexp = /\r\n|\r|\n/;
      const noNewlineTexts = text.split(newlineRegexp);

      let temp = "";
      noNewlineTexts.forEach((str, index) => {
        temp += str += "\r\n"; //각 배열에 newline을 추가해줌
      });

      const blob = new Blob([temp], {
        type: "text/plain;charset=utf-8"
      });

      saveAs(blob, "Voca.txt");
    }
  }
};
</script>
