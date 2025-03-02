<script lang="ts">
	import { onMount } from 'svelte';
    import {CLIENT_ID,SMART_AUTH_URL, REDIRECT_URI, FHIR_BASE_URL, SMART_TOKEN_URL} from '../config'
    import axios from 'axios';
    const generateRedirectUrl = () => {
        const authorizationUrl = new URL(SMART_AUTH_URL)
        authorizationUrl.searchParams.set('client_id', CLIENT_ID)
        authorizationUrl.searchParams.set('scope', 'openid fhirUser')
        authorizationUrl.searchParams.set('redirect_uri', REDIRECT_URI)
        authorizationUrl.searchParams.set('response_type', 'code')
        authorizationUrl.searchParams.set('state', '1234567')
        authorizationUrl.searchParams.set('aud', FHIR_BASE_URL)

       authorizationUrl.searchParams.set('code_challenge','')
       authorizationUrl.searchParams.set('code_challenge_method','S256')

        return authorizationUrl.href

    }

    onMount(async () => {
        const code = new URL(window.location.href).searchParams.get('code')
        console.log(code)
        if(code){
            await makeTokenRequest(code)
        }
    })

    const makeTokenRequest = async (code: string) => {
        const tokenRequestForm = new FormData();
        tokenRequestForm.set('grant_type', 'authorization_code')
        tokenRequestForm.set('code', code)
        tokenRequestForm.set('redirect_uri', REDIRECT_URI)
        tokenRequestForm.set('client_id', CLIENT_ID)

        const response = await axios.postForm(SMART_TOKEN_URL, tokenRequestForm)
        console.log(response)
    }
</script>
<main>
    <div class="flex justify-center my-20"> <button class="p-3 bg-black text-white" on:click={()=>{console.log(generateRedirectUrl())}}>Sign in with Epic</button></div>
   
</main>