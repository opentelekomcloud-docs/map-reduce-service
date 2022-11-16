:original_name: mrs_02_1080.html

.. _mrs_02_1080:

API Calling Process
===================

The process for calling an MRS Manager API is as follows:

#. :ref:`Obtaining Request Authentication Information <mrs_02_1080__en-us_topic_0125376234_section1225494452418>`

   Before you call an API, obtain request authentication information, and fill it in a request header.

#. :ref:`Obtaining Request Authentication Information <mrs_02_0008>`

   Configure request parameters to construct a request.

#. :ref:`Sending a Request <mrs_02_0008>`

#. :ref:`Parsing a Response <mrs_02_0010>`

.. _mrs_02_1080__en-us_topic_0125376234_section1225494452418:

Obtaining Request Authentication Information
--------------------------------------------

**Scenario**

Currently, CAS can be used for login authentication when an API is called.

After login authentication, obtain the JSESSIONID of a web request and check the current user using the JSESSIONID.

**Procedure for Calling an API** (Security Cluster)

#. Send the request **GET https://**\ **MRS Managerfloating IP**\ **:20009/cas/login** to obtain **casSessionId** and **loginTicket** from the CAS login information.

   The following example describes how to obtain **casSessionId** and **loginTicket** from the response object of the request.

.. code-block::

   // Obtain the CAS login page.
   HttpResponse casLoginPageResponse = getCasLoginPage(casUrl, httpClient);
   LOG.info("get cas login page request status is :{} , casLoginPageResponse is :{}.", casLoginPageResponse.getStatusLine(), casLoginPageResponse);

   // Obtain casSessionId and loginTicket.
   String casSessionId = getCasSessionId(casLoginPageResponse);
   String loginTicket = getLoginTicket(casLoginPageResponse);
   LOG.info("casSessionId = {} , loginTicket = {}.", casSessionId, loginTicket);

2. Send **POST https://**\ *MRS Manager floating IP*\ **:20009/cas/login?service= https://**\ *MRS Manager floating IP*\ **:28443/web/cas_security_check.htm** to initiate a CAS login authentication request. The request body contains the username, password, and the obtained **loginTicket**. The request header contains **casSessionId**. Obtain the authenticated ticket granting cookie (TGC) from the response header.

.. code-block::

   // Authenticate the username and password on the CAS server.
   HttpResponse loginPostResponse =
           loginCasServer(casUrl, webUrl, userName, password, httpClient, casSessionId, loginTicket, userTLSVersion);
   LOG.info("the post response is: {}.", loginPostResponse);

   // Obtain the authenticated TGC.
   String casTgc = getCASTGC(loginPostResponse);
   LOG.info("casTgc = {}.", casTgc);
   ...


   /**
    * Log in to the CAS server and verify user information.
   * @param casUrl CAS server address
   * @param webUrl Web application address
    * @param userName Login user name
    * @param password Login password
    * @param casSessionId CAS session ID
    * @param loginTicket CAS login ticket
    * @return http client HTTP client
    */
   private HttpResponse loginCasServer(String casUrl, String webUrl, String userName, String password,
       HttpClient httpClient, String casSessionId, String loginTicket,String userTLSVersion)
   {
       if (ParamsValidUtil.isEmpty(casUrl, webUrl, userName, password, casSessionId, loginTicket))
       {
           LOG.error("Invalid input param.");
       }
       String postUrl = generateCasLoginUrl(casUrl, webUrl);
       LOG.info("login cas server URL is : {}.", postUrl);

       HttpPost httpPost = new HttpPost(postUrl);
       // Parameters contained in the request body
       List<BasicNameValuePair> FormData = new ArrayList<BasicNameValuePair>();
       FormData.add(new BasicNameValuePair("username", userName));
       FormData.add(new BasicNameValuePair("password", password));
       FormData.add(new BasicNameValuePair("lt", loginTicket));
       FormData.add(new BasicNameValuePair("_eventId", "submit"));
       FormData.add(new BasicNameValuePair("submit", "Login"));
       HttpResponse response = null;
       BufferedReader bufferedReader = null;
       // Parameters contained in the request header
       httpPost.addHeader("Cookie", CAS_SESSION_ID_STRING + casSessionId);
   ...


   /**
    * Generate a complete URL for logging in to CAS.
    * After the login is successful, the specified CAS security check page is displayed.
    * @param casUrl
    * @param webUrl
    * @return
    */
   private String generateCasLoginUrl(String casUrl, String webUrl)
   {
       StringBuilder sb = new StringBuilder();
       sb.append(casUrl);
       sb.append("?service=");
       sb.append(webUrl);
       sb.append("/cas_security_check.htm");
       return sb.toString();
   }

3. Send **GET https://**\ *MRS Manager floating IP*\ **:28443/web/v1/access/login_check** to initiate a web login authentication request. The cookie in the request header contains two TGC obtained in the previous step. After the response is successful, obtain the JSESSIONID of the web request from the cookie in the response header.

.. code-block::

   /**
    * Initiate a request for web application login authentication.
    * @param webUrl Web application URL
    * @param httpclient http client
    * @return response Login verification response object
    */
   private HttpResponse webLoginCheck(String webUrl, HttpClient httpclient)
   {
       //Web login authentication request path
       HttpGet loginCheckHttpGet = new HttpGet(webUrl + "/v1/access/login_check");
       LOG.info("web login check URL is: {}.", (webUrl + "/v1/access/login_check"));

       HttpResponse response = null;
       BufferedReader bufferedReader = null;
       InputStream inputStream  = null;
       boolean flag = false;
       try
       {
           response = httpclient.execute(loginCheckHttpGet);

           inputStream = response.getEntity().getContent();
           bufferedReader = new BufferedReader(new InputStreamReader(inputStream));
           String lineContent = "";
           lineContent = bufferedReader.readLine();
           LOG.info("response content is {} : " + lineContent);
           String postResponseState = "";
   ...

.. note::

   For a non-security cluster, you can skip CAS authentication in step 1 and step 2 and directly send the **GET https://**\ **MRS Manager floating IP**\ **:28443/web/v1/access/login_check** web login authentication request. And then, you can obtain the **FISessionId** of the web request from the cookie of the response header of the response object.
