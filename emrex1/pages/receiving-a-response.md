Receive a response
==================

[Technical guide](technical-guide.md)

The response is sent as a HTTP POST back to the EMREX Client with four parameters.

The following return codes are supported (the list is subject to change):

1. NCP_OK: Everything wen well, results have been transferred

2. NCP_ERROR: Something went wrong. The error will be in the “returnMessage”

3. NCP_NO_RESULTS: There were no results to import into the client.

4. NCP_CANCEL: The user has cancelled

The “sessionId” must be the same as the one sent in the request. If it’s not the same as the one that was sent, this response should not be processed.

The “ELMO” parameter is the main piece of this response. It will be gzipped and encoded in Base64 for transfer.

[Choosing the EMP](choosing-the-emp.md)

[Sending a request to the EMP](sending-a-request-to-the-emp.md)

[Interpreting and handling the data](interpreting-and-handling-the-data.md)