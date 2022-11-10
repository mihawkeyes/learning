<template>
    <div>
        <div id="app" style="position: relative;">
            <template>
                <template v-if="intruder">
                    <Modal
                        v-if="intruder"
                        type="danger"
                        heading="Siloho"
                        title="Something went wrong"
                        desc="Please try again later or contact your provider"
                    />
                </template>
                <template v-else>
                    <NavBar
                        :currentIndex="currentIndex"
                        :show="
                        activeComponent == 'ResultContainerWrapper' ||
                        activeComponent == 'StartQuiz' ||
                        activeComponent == 'PreEHD' ||
                        currentIndex == 0
                            ? false
                            : true
                        "
                        :prev="previousQuestion"
                        :cross="showclose"
                        :builder_id="builder_id"
                        @previousQuiz="previousQuiz"
                        @closeFrame="close"
                    />

                    <!-- component required for quiz is dynamically mounted using this tag -->
                    <component
                        v-bind:is="activeComponent"
                        :builder_id="builder_id"
                        :question="question"
                        :totalQuestions="totalQuestion"
                        :totalImagesRequired="totalImageCount"
                        :completedQuestions="answeredQuestion"
                        :currentIndex="currentIndex"
                        :calculatedResult="calculatedResult"
                        :windowWidth="windowWidth"
                        :ehd_id="ehd_id"
                        :theme_id="
                        Object.keys(calculatedResult).length != 0
                            ? calculatedResult.theme_id
                            : null
                        "
                        :displayType="resultDisplayType"
                        @startQuiz="startQuiz"
                        @next="nextQuiz"
                        @retakeQuiz="retakeQuiz"
                        @loadMoreImages="loadMoreImages"
                        :answer="answer"
                        :surveyResponseId="survey_response_id"
                        :developerUuid="surveyResponse.developer_uid!=undefined?surveyResponse.developer_uid:''"
                        :tempUserUuid="tempUserUuid!=''?tempUserUuid:''"
                        :autoLoadedImages="autoLoadedImages"
                        :themeImageAutoLoad="themeImageAutoLoad"
                    />

                    <FullModal
                        v-if="isModalVisible"
                        @close="closeModal"
                        setTransparent
                        :showClose="showModalClose"
                        :prefill="prefilledLogin"
                    >
                        <template v-slot:default="slotProps">
                            <SignupBox
                                v-if="authChange"
                                @auth="authenticate"
                                @changeAuth="toggleAuthChange(!authChange)"
                                :hideBack="slotProps.hideBack"
                                :hideCloseIcon="hideCloseIcon"
                                :closeModal="closeModal"
                                :back="slotProps.back"
                            />
                            <LoginBox
                                v-else
                                @auth="authenticate"
                                @changeAuth="toggleAuthChange(!authChange)"
                                :hideCloseIcon="hideCloseIcon"
                                :closeModal="closeModal"
                                :hideBack="slotProps.hideBack"
                                :back="slotProps.back"
                                :prefill="prefilledLogin"
                                :number="rnumber"
                                :oauth="oauthID"
                                :email="emailID"
                                :loginWithOauth="oauthFlag"
                            />
                        </template>
                    </FullModal>

                    <FullModal v-if="errorModalVisible" @close="closeErrorModal">
                        <div class="errormodal">
                            <div class="errormodal-view">
                                <img
                                    class="errormodal-img"
                                    src="@/assets/images/thankyou.png"
                                />
                                <h4>Thank You</h4>
                                <p>Our team will Connect within 2 Working days.</p>
                                <p>You can always reach out on this number</p>
                                <p>+91 8-805-806-805</p>
                                <a :href="`${ecomUrl}designers`">
                                <Button
                                    class="explore-designs"
                                    buttonName="Explore Designers"
                                    primary
                                />
                                </a>
                            </div>
                        </div>
                    </FullModal>
                </template>
            </template>

            <!-- Iframe for VT widget -->
            <iframe
                v-if="widget_type == 'e'"
                id="embededvirtualtour"
                style="display:none;"
                :src="calculatedResult.vt_url ? calculatedResult.vt_url : ''"
            ></iframe>

            <!------------------------TESTING DYNAMIC COMPONENTS INDIVIDUALLY--------------------------->
            <template v-if="resultDisplayType == 'PANORAMICRESULT'">
            </template>
            <template v-else-if="resultDisplayType == 'DECOUPLERESULT'">
            </template>
            <template v-else>
                <Footer
                    v-if="survey_completed != null"
                    :show="activeComponent == 'ResultContainerWrapper' ? true : false"
                    :selectedImageCount="question.question_type == 'TH'?true:false"
                    :vtlink="
                        Object.keys(calculatedResult).length != 0
                        ? calculatedResult.vt_url
                        : ''
                    "
                    :widget_type="widget_type"
                    :theme_id="
                        Object.keys(calculatedResult).length != 0
                        ? calculatedResult.theme_id
                        : null
                    "
                    :EHD="EHD"
                    :builder_id="builder_id"
                    :ehd_id="ehd_id"
                    @displayIframe="displayIframe"
                    :pre-ehd="preEhd"
                    @ehdRedirect="goToEhd"
                    :disabledbtn="disabledbtn"
                    :totalImageCount="totalImageCount"
                />
            </template>
        </div>
        <template v-if="loaderShow">
            <div class="wait">
                <Loader :msg="msg" />
            </div>
        </template>
        <template v-if="visualLoaderShow">
            <div class="wait">
                <VisualLoader />
            </div>
        </template>
    </div>
</template>

<script>
import { IS_LOGGEDIN, DOMAIN_NAME, SQ, PQ, EMAIL } from "utils/constants"
import { bus } from "main"
// this all components currently in use
import Modal from "components/Modal"
import FullModal from "components/FullModal"
import SignupBox from "components/SignupBox"
import LoginBox from "components/LoginBox"
import Button from "components/Button"
import Spinner from "components/Spinner"
import NavBar from "components/NavBar"
import StartQuiz from "components/StartQuiz"
import QuestionContainer from "components/QuestionContainer"
import Footer from "components/Footer"
import RadioButton from "components/RadioButton"
import Select from "components/Select"
import PriceRange from "components/PriceRange"
import ProjectDetails from "components/ProjectDetails"
import ResultContainerWrapper from "components/ResultContainerWrapper"
import Loader from "components/Loader"
import PreEHD from "components/PreEHD"
import VisualLoader from "components/VisualLoader"
// import SQResult from "components/SQResult"

import Cookies from "universal-cookie"

const cookies = new Cookies()

export default {
    name: "App",
    components: {
        Modal,
        FullModal,
        SignupBox,
        LoginBox,
        Button,
        Spinner,
        NavBar,
        StartQuiz,
        QuestionContainer,
        Footer,
        RadioButton,
        PriceRange,
        ProjectDetails,
        Select,
        ResultContainerWrapper,
        Loader,
        PreEHD,
        VisualLoader
    },
    data() {
        return {
            styleQuizUrl: process.env.VUE_APP_BASE_URL,
            ecomUrl: process.env.VUE_APP_ECOM_BASE_URL,
            windowWidth: window.innerWidth,
            intruder: false,
            builder_id: null,
            ehd_id: null,
            loading: true,
            widget_type: "",
            showclose: false,
            parentUrl: null,
            verified: false,
            activeComponent: "StartQuiz",
            surveyType: "",
            survey_id: null,
            survey_completed: null,
            survey_response_id: null,
            currentIndex: -1,
            question: {},
            previousQuestion: null,
            totalQuestion: 0,
            answeredQuestion: 0,
            surveyResponse: {},
            EHD: false,
            calculatedResult: {},
            resultDisplayType: null,
            isModalVisible: false,
            errorMsg: "",
            errorModalVisible: false,
            email: null,
            isLoggedIn: cookies.get(IS_LOGGEDIN)
                ? JSON.parse(cookies.get(IS_LOGGEDIN))
                : false,
            authChange: true,
            showModalClose: true,
            answer: {},
            userDate: null,
            loaderShow: false,
            msg: "Understanding requirements",
            preEhd: false,
            loginRequired: null,
            disabledbtn: null,
            prefilledLogin: false,
            rnumber: "",
            tempUserUuid: "",
            visualLoaderShow:false,
            oauthID:"",
            emailID:"",
            oauthFlag:null,
            //when autoload flag is true api will be called to load more images, if it is false then api will not be called
            autoLoad:true,
            //imagePageNo will refer to the pages that will be requested via api
            imagePageNo:1,
            autoLoadedImages:[],
            totalImageCount:0,
            themeImageAutoLoad:true,
        }
    },
    watch: {
        isLoggedIn: function() {
            if(this.isLoggedIn==true && this.surveyType=="SQ"){
                this.getResult()
            }

            if(this.isLoggedIn==true && this.surveyType=="PQ"){
                this.getSurvery(this.survey_id)
                this.disabledbtn = true
            }
        },

        survey_completed: function(){
            if(this.survey_completed == true){
                this.nextQuiz()
            }
        }
    },
    created() {
        // check for setResponse emit by child components
        window.addEventListener("resize", () => {
            this.windowWidth = window.innerWidth
        })

        bus.$on("setResponse", (data) => {
            this.surveyResponse.developer_uid = this.builder_id != null ? this.builder_id : ""
            this.surveyResponse.question_id = data.question_id
            this.surveyResponse.response_choices = data.response_choices
        })

        bus.$on("nextQuiz", () => {
            this.nextQuiz()
        })
    },
    mounted() {
        this.builder_id = this.$route.params.u_id ? this.$route.params.u_id : null
        this.widget_type = this.$route.params.widget_type ? this.$route.params.widget_type : null

        if (this.$route.query.startQuiz == "true") {
            this.startQuiz(true)
            this.activeComponent = ""
        }

        // check if the style quiz is used in iframe
        let isInIframe = parent !== window

        let link = document.referrer
        this.parentUrl = link.substring(0, link.length - 1)

        if (isInIframe) {
            //this.loading = false
            // IFrame builder verification
            this.verifyBuilder(this.parentUrl, this.builder_id)
        } else {
            //if not used in frame check parameter of url if parameter exist on non iframe link then it is intruder
            //this.loading = false
            if (this.$route.params.widget_type && this.$route.params.u_id) {
                this.intruder = true
                return
            }
        }
    },
    beforeDestroy() {
        // bus.$off('setResponse')
        // bus.$off('nextQuiz')
    },
    methods: {
        showModal() {
            this.isModalVisible = true
        },

        closeModal() {
            this.disabledbtn = false
            this.isModalVisible = false
        },

        hideCloseIcon(status) {
            this.showModalClose = status
        },

        closeErrorModal() {
            this.errorModalVisible = false
        },

        verifyBuilder(parentUrl, builder_id) {
            /**
                Make api call to get builder website link from database to cross check it with   
                iframe host url
            **/
            this.$http
            .get(`/RealEstateManagement/developer/retrieve_by_uid/?developer_uid=${builder_id}`)
            .then((res) => {
                const developer_data = res.data.developer_data
                //this.loading = false
                if (parentUrl !== developer_data.link) {
                    this.intruder = true
                    this.verified = true
                    return
                }
                this.verified = true
            })
            .catch(() => {
                this.intruder = true
                // this.loading = false
            })
        },

        //This function is used to implement infinite scroll for the images used on style quiz
        loadMoreImages(){      
            if(this.autoLoad==true){
                this.imagePageNo++
                this.autoLoad=false
                this.$http.post(`v2/quiz/survey-response/${this.survey_response_id}/survey_submit/`, {
                    page:this.imagePageNo,
                    size:20,
                    question_id:this.question.id,
                    developer_uid:  this.builder_id != null ? this.builder_id : "",
                    userCookies: this.tempUserUuid
                })
                .then((response)=>{
                    let res = response.data
                    if (res.responseCode == 200) {
                        if(this.imagePageNo<=res.payload.next_question.total_pages){
                            this.autoLoadedImages = res.payload.next_question.choice_data
                            this.autoLoad=true
                        }else{                                
                            this.themeImageAutoLoad=false
                            this.autoLoad=false
                            this.autoLoadedImages=null
                        }
                      
                    } else {
                        console.log(res.responseCode, res.responseMessage)
                    }
                    
                })
            }
        },
        // change start style quiz button behavior based on widget or not widget
        startQuiz() {
            
            if (this.$route.params.widget_type == "r") {
                window.open(this.styleQuizUrl + `startQuiz=true&u_id=${this.builder_id}`)
            } else if (this.$route.params.widget_type == "m") {
                let body_params = {}
                if (this.builder_id != null) {
                    body_params.developer_uid = this.builder_id
                }
                this.getCreateProject(body_params);
            } else if (this.$route.params.widget_type == "e") {
                window.parent.postMessage("expand", this.parentUrl)
                this.showclose = true
                let body_params = {}
                if (this.builder_id != null) {
                    body_params.developer_uid = this.builder_id
                }
                this.getCreateProject(body_params)
            } else {
                this.EHD = true
                let body_params = {
                    developer_uid: "",
                }
                body_params.survey_id = this.$route.query.s ? this.$route.query.s : ""
                body_params.re_take_survey = this.$route.query.fq ? this.$route.query.fq : false
                if (this.$route.query.u_id) {
                    this.EHD = false
                    this.builder_id = this.$route.query.u_id
                    body_params.developer_uid = this.$route.query.u_id
                        ? this.$route.query.u_id
                        : ""
                    this.getCreateProject(body_params)
                } else {
                    this.getCreateProject(body_params)
                }
            }
        },

        getCreateProject(params) {
            this.$http
            .post(`/quiz/survey/create_project/`, params)
            .then((response) => {
                let res = response.data
                if (res.responseCode == 200) {
                    this.loginRequired = res.payload.login_required
                    this.surveyType = res.payload.survey_type
                    this.survey_id = res.payload.action_id
                    this.tempUserUuid = res.payload.user_cookies
                    if(this.loginRequired==true && this.surveyType=="PQ" && this.isLoggedIn==false){
                        this.isModalVisible = true
                        this.prefilledLogin = false
                        this.authChange = false
                    }else{
                        this.action(res.payload)
                    }
                } else {
                    console.log(res.responseCode, res.responseMessage)
                }
            })
            .catch(() => {
                console.log("Some issue occured please try again later")
            })
        },

        //if survey is given already redirect to EHD or MUDI (given action page)
        action(data) {
            switch (data.action) {
                case "EHD":
                    window.open(
                        `${this.ecomUrl}${data.action.toLowerCase()}/${data.action_id}`,
                        "_self"
                    )
                break
                case "MUDI":
                    window.open(
                        `${this.ecomUrl}${data.action.toLowerCase()}/designideas`,
                        "_self"
                    )
                break
                default:
                    this.getSurvery(data.action_id)
                break
            }
        },

        getSurvery(params) {
            var body = {}
            if(this.tempUserUuid!=""){
                body.developer_uid = this.builder_id,
                body.userCookies = this.tempUserUuid
            }
            body.re_take_survey = this.$route.query.fq ? this.$route.query.fq : false
            this.$http
            .post(`v2/quiz/survey/${params}/take_survey/`,body)
            .then((response) => {
                let res = response.data
                if (res.responseCode == 200) {
                    this.survey_completed = res.payload.survey_completed
                    this.survey_response_id = res.payload.survey_response_id
                    this.question = res.payload.next_question
                    this.activeComponent = this.survey_completed == false ? "QuestionContainer" : "StartQuiz"
                    this.answer = {}
                    this.totalQuestion = res.payload.max_survey_questions
                    this.answeredQuestion = res.payload.survey_questions_answered
                    this.previousQuestion = res.payload.previous_question_available
                    if(res.payload.next_question.min_selections){
                        this.totalImageCount=res.payload.next_question.min_selections
                    }
                } else {
                    console.log(res.responseCode, res.responseMessage)
                }
            })
            .catch((e) => {
                console.log(e)
                console.log("Some issue occured please try again later")
            })
        },

        getResult() {
            var body
            if(this.tempUserUuid!=""){
                body={
                    developer_uid: this.builder_id,
                    userCookies: this.tempUserUuid
                }
            }
            this.$http.post(`v2/quiz/survey-response/${this.survey_response_id}/survey_result/`,body)
            .then((response) => {
                this.previousQuestion = false
                let res = response.data
                const now = new Date()
                now.setTime(now.getTime() + 30 * 24 * 60 * 60 * 1000)
                if(res.is_sq_completed){
                    cookies.set(SQ, res.is_sq_completed, {
                        path: "/",
                        domain: DOMAIN_NAME,
                        expires: now,
                    })
                }
                if(res.is_pq_completed){
                    cookies.set(PQ, res.is_pq_completed, {
                        path: "/",
                        domain: DOMAIN_NAME,
                        expires: now,
                    })
                }
                if (res.responseCode == 200) {
                    this.previousQuiz = {}
                    if (res.payload.survey_type == "PQ") {
                        switch (res.payload.landing_page) {
                            case "EHD":
                                this.preEhd = true
                                this.activeComponent = "PreEHD"
                                this.ehd_id = res.payload.id
                                this.$http.post(`CADVisualization/get_pq_lending_data/`, {
                                    ehd_id: this.ehd_id,
                                }).then((response) => {
                                    let res = response.data
                                    this.calculatedResult = res.payloads
                                })
                            break
                            case "MUDI":
                                this.loaderShow = true
                                document.querySelector("body").style.overflow = "hidden"
                                setTimeout(() => {
                                    this.msg = "Looking up designs"
                                    setTimeout(() => {
                                        this.msg = "Creating workspace"
                                        window.open(
                                            `${this.ecomUrl}${res.payload.landing_page.toLowerCase()}/designideas`,
                                            "_self"
                                        )
                                    }, 1000)
                                }, 1500)
                            break
                        }
                    } else {
                        this.activeComponent = "ResultContainerWrapper"
                        this.calculatedResult = res.payload
                        this.ehd_id = this.calculatedResult.ehd_id
                        this.resultDisplayType = this.calculatedResult.landing_page
                    }
                } else {
                    console.log(res.responseCode, res.responseMessage)
                }
            })
            .catch(() => {
                console.log("Some issue occured please try again later")
            })
        },

        getNextQuestion(){
            if(this.tempUserUuid!=""){
                this.surveyResponse.userCookies = this.tempUserUuid
            }
            this.$http.post(`v2/quiz/survey-response/${this.survey_response_id}/survey_submit/`, this.surveyResponse)
            .then((response)=>{
                let res = response.data
                if (res.responseCode == 200) {
                    this.survey_completed = res.payload.survey_completed
                    this.survey_response_id = res.payload.survey_response_id
                    if(res.payload.next_question!=null){
                        this.question = res.payload.next_question
                        if(res.payload.next_question.min_selections){
                            this.totalImageCount=res.payload.next_question.min_selections
                        }
                    }
                    this.activeComponent = "QuestionContainer"
                    this.answer = {}
                    this.totalQuestion = res.payload.max_survey_questions
                    this.answeredQuestion = res.payload.survey_questions_answered
                    this.previousQuestion = res.payload.previous_question_available
                } else {
                    console.log(res.responseCode, res.responseMessage)
                }
            })
        },

        // goes forward in questions array in data property
        nextQuiz() {
            this.disabledbtn = null
            if (this.survey_completed == true) {
                if (this.loginRequired == true && this.isLoggedIn == false && this.surveyType=="SQ") {
                    this.showModal()
                    if(cookies.get(EMAIL)){
                        //this.getNextQuestion()
                        this.checkCustomerExist(cookies.get(EMAIL))
                        
                    }
                } else {
                    this.getResult()
                }
            } else {
               this.getNextQuestion()
            }
        },

        // goes backward in questions array in data property
        previousQuiz() {
            var body = {}
            if(this.tempUserUuid!=""){
                body.developer_uid = this.builder_id,
                body.userCookies = this.tempUserUuid
            }
            body.question_id = this.question.id
            this.$http.post(`v2/quiz/survey/previous_question/`, body)
            .then((response)=>{
                let res = response.data
                if (res.responseCode == 200) {
                    this.question = res.payload.previous_question
                    this.previousQuestion = res.payload.previous_question_available
                } else {
                    console.log(res.responseCode, res.responseMessage)
                }
            })
        },

        // this function take user to back to start page
        retakeQuiz() {
            if (this.$route.query.startQuiz === "true") {
                this.startQuiz(true)
                this.activeComponent = ""
            } else {
                this.activeComponent = "StartQuiz"
            }

            this.currentIndex = -1
            this.question = {}
            this.calculatedResult = {}
            this.survey_completed = null
        },

        close() {
            window.parent.postMessage("close", this.parentUrl)
            this.showclose = false
            this.retakeQuiz()
            document.getElementById("embededvirtualtour").style.cssText = "display:none;"
            document.querySelector("body").style.cssText = "overflow: '';"
        },

        // if parent is iframe instead of redirecting include iframe for virtualtour
        displayIframe() {
            document.getElementById("embededvirtualtour").style.cssText =
                "position: fixed; top: 50px; left :0; height: 92vh; width: 100%; z-index: 1000; border: none;"
            document.querySelector("body").style.cssText = "overflow: hidden;"
        },

        authenticate(e) {
            this.isLoggedIn = JSON.parse(e)   
        },

        toggleAuthChange(param) {
            this.authChange = param
        },

        getAnswer() {
            if (this.survey.answers) {
                this.survey.answers.data.forEach((x) => {
                    if (x.question_id == this.questions[this.currentIndex].id) {
                        this.answer = x
                    }
                })
            }
        },

        goToEhd() {
            this.visualLoaderShow = true

            setTimeout(() => {
                window.open(`${this.ecomUrl}ehd/${this.ehd_id}`, "_self")
                this.visualLoaderShow=false
            }, 8000)
            // setTimeout(() => {
            //     this.msg = "Looking up designs"
            //     setTimeout(() => {
            //         this.msg = "Identifying your dream home"
            //         setTimeout(() => {
            //             window.open(`${this.ecomUrl}ehd/${this.ehd_id}`, "_self")
            //         }, 500)
            //     }, 1000)
            // }, 1500)
        },

        checkCustomerExist(email) {
            this.$http.post(`usermanagement/users/customer_check/`, {
                email_id: email,
            })
            .then((res) => {
                let response = res.data
                if (response.responseCode == 2003) {
                    if(response.payload.phone_number!=""){
                        this.rnumber = response.payload.phone_number
                        this.oauthFlag=false
                    }else{
                        this.oauthFlag=true
                        this.oauthID=response.payload.oauth_id
                        this.emailID=email
                    }
                    this.prefilledLogin = true
                    this.toggleAuthChange(false)
                    
                } else {
                    this.prefilledLogin = false
                    this.toggleAuthChange(true)
                }
            })

        },
    },
}
</script>

<style lang="scss">
@import "./App.scss";
</style>
