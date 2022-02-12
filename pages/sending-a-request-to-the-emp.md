Sending a request to the EMP
============================

[Technical guide](technical-guide.md)

The request is sent from a requester as HTTP POST and has two parameters. Note that the form must be of the default type, not “multiform/form-data”:

The hidden parameters are as follows:

1. sessionId: A generated unique session id from the requester. This session id is not used by the EMP but us returned as part of the reply so that the requester can check that the response comes from the EMP as past of the same session.

2. returnUrl: The URL that the EMP shall use to post the reply.

As this service is still under development, additional parameters might be added at a later point.

[Choosing the EMP](choosing-the-emp.md)

[Receiving a response](receiving-a-response.md)

[Interpreting and handling the data](interpreting-and-handling-the-data.md)