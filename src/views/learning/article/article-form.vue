<!--
    views/learning/article/article-form.vue 기능 설명

    board.vue(게시판 메인)에서 글 생성버튼을 누르면 렌더링 되는 페이지 코드이다.
    즉, 실질적인 강의자료나 게시물들을 작성하는 form 이다. ( toast editor (위지윅 모드) 사용)

    이 컴포넌트 안에서 게시물 최초 작성과 수정을 할 수 있게끔 하였다. (v-if 이용)
-->

<template>
  <v-container fluid>
    <v-form>
      <v-card :loading="loading">
        <v-toolbar color="accent" dense flat dark>
          <v-toolbar-title>게시물 작성</v-toolbar-title>
        <v-spacer/>
        <v-btn icon @click="$router.push('/'+collection+ '/' + document)"><v-icon>mdi-arrow-left</v-icon></v-btn> <!-- 뒤로가기 -->
        <v-btn icon @click="save" :disabled="!valid"><v-icon>mdi-content-save</v-icon></v-btn>
        </v-toolbar>
        <v-card-text>
          <v-form v-model="valid" ref="form">
          <v-text-field v-model="form.title" outlined label="게시물 제목" required :rules="[rule.required]"></v-text-field>
          <v-container fluid>
            <v-row align="center">
              <v-col class="d-flex" cols="4" sm="2">
                <v-select v-model="form.year" :items="yearList" label="년" required :rules="[rule.required]"></v-select>
              </v-col>
              <v-col class="d-flex" cols="4" sm="2">
                <v-select v-model="form.month" :items="monthList" label="월" required :rules="[rule.required]"></v-select>
              </v-col>
              <v-col class="d-flex" cols="4" sm="2">
                <v-select v-model="form.week" :items="weekList" label="주차" required :rules="[rule.required]"></v-select>
              </v-col>
            </v-row>
            </v-container>
            </v-form>

          <v-toolbar-title class="mt-1">본문 작성</v-toolbar-title>
          <!-- articleId 가 없다는 것은 글 최초 작성 -->
          <editor v-if="!articleId" initialEditType='wysiwyg' :options="editor_options" :initialValue="form.content" ref="editor"></editor>
          <!-- aritcleId 가 있다는 것은 글 수정, 따라서 기존의 title 과 content 를 렌더링해줘야한다. -->
          <template v-else>
            <editor v-if="form.content" initialEditType='wysiwyg' :options="editor_options" :initialValue="form.content" ref="editor"></editor>
            <!-- v-else 로 들어온다는 것은 아직 form.content에 값이 안들어온 것(axios로 데이터 가져오는데 시간필요)이므로 로딩 ui 생성 -->
            <v-container v-else>
              <v-row justify="center" align="center">
                <v-progress-circular indeterminate></v-progress-circular>
              </v-row>
            </v-container>
          </template>
          <v-spacer/>
          <v-container v-if="document === 'jungsin'"> <!-- 공지사항과 우리역사바로알기에는 렌더링 안함 -->
          <v-toolbar-title class="mt-5">평가 질문 작성</v-toolbar-title>
          <v-text-field v-model="form.Q1" outlined label="질문1"></v-text-field>
          <v-text-field v-model="form.Q2" outlined label="질문2"></v-text-field>
          </v-container>
        </v-card-text>
      </v-card>
    </v-form>
  </v-container>
</template>
<script>
import axios from 'axios'
export default {
	props: ['collection', 'document', 'action'],
	data () {
		return {
			yearList: ['2020년', '2021년'],
			monthList: ['1월', '2월', '3월', '4월', '5월', '6월', '7월', '8월', '9월', '10월', '11월', '12월'],
			weekList: ['1주차', '2주차', '3주차', '4주차', '5주차'],
			editor_options: {
				language: 'ko',
				hideModeSwitch: true
			},
			form: {
				title: '',
				content: '', // content 는 이미지,동영상 같은 파일들이므로 Storage 에 넣어야 함
				Q1: '',
				Q2: '',
				month: '',
				week: '',
				year: ''
			},
			exists: false,
			loading: false,
      ref: null,
      valid: false, // form valid
      rule: { required: v => !!v || '필수 항목입니다.' }
		}
	},
	computed: {
		articleId () {
			return this.$route.query.articleId // articleId 는 board.vue 에서 넘어온 쿼리값 (수정할때만 id 값이 넘어온다)
		},
		user () { // Vuex state에 저장돼있는 user 정보
			return this.$store.state.user
		}
	},
	watch: { // document 값이 바뀔 때, fetch() 함수를 실행시켜서 일회성으로 DB 에서 데이터 받아옴
		document () {
			this.fetch()
		}
	},
	created () {
		this.fetch()
	},
	methods: {
		async fetch () {
			this.ref = this.$firebase.firestore().collection(this.collection).doc(this.document)

			// articleId 쿼리값이 없을 때 ( 글 최초작성)
			if (!this.articleId) return

			// articleId 쿼리값이 있을 때 ( 글 수정)
			const doc = await this.ref.collection('articles').doc(this.articleId).get()
			this.exists = doc.exists
			if (!this.exists) return // doc 가 없으면 종료
			const item = doc.data() // 임시변수 item 사용
			this.form.title = item.title
			this.form.Q1 = item.question.Q1
			this.form.Q2 = item.question.Q2
			const r = await axios.get(item.url)
			this.form.content = r.data
			this.form.month = item.month
			this.form.week = item.week
			this.form.year = item.year
		},
		async save () { // 작성한 글 저장 함수 : 비동기 로직 포함 ( firestore DB에 저장 )
			if (this.user.level !== 'admin') throw Error('관리자만 가능합니다!') // 권한 확인
			this.loading = true
			try {
				const now = new Date()
				const content = this.$refs.editor.invoke('getHtml') // 에디터에서 작성한 글 (html 파일로 변환)
				const doc = {
					title: this.form.title,
					updatedAt: now,
					url: '',
					question: {
						Q1: this.form.Q1,
						Q2: this.form.Q2
					},
					month: this.form.month,
					week: this.form.week,
					year: this.form.year
				}

				const batch = await this.$firebase.firestore().batch()

				if (!this.articleId) { // 새로 작성할 때
					doc.article_id = now.getTime().toString() // 작성시간을 id로 사용 ( computed 에 있는 articleId 와는 다름! )
					doc.createdAt = now
					doc.uid = this.user.uid // 수정되면 안되는 정보이므로 일부러 새로 작성할 때만 정보 생성
					const sn = await this.$firebase.storage().ref().child(this.collection).child(this.document).child(doc.article_id + '.html').putString(content)
					const url = await sn.ref.getDownloadURL()
					doc.url = url
					batch.set(this.ref.collection('articles').doc(doc.article_id), doc)
					batch.update(this.ref, { count: this.$firebase.firestore.FieldValue.increment(1) })
				} else { // 기존 게시물을 수정할 때
					const sn = await this.$firebase.storage().ref().child(this.collection).child(this.document).child(doc.article_id + '.html').putString(content)
					const url = await sn.ref.getDownloadURL()
					doc.url = url
					batch.update(this.ref.collection('articles').doc(this.articleId), doc)
				}
				await batch.commit()
			} finally {
				this.loading = false
				this.$router.push('/' + this.collection + '/' + this.document)
			}
		}
	}
}
</script>
