Choosing the EMP
================

[Technical guide](technical-guide.md)

In order to initiate the transfer, one must first choose the National Contact Point to get the results from. This is done by contacting EMREG, a centralized service that gives a list of all available EMPs, as well as other information necessary to establish communication with each of them.

At the moment, the URL for EMREG is as follows;

– Test: https://fsweb-demo.uio.no/emreg/list/test

– Production: https://fsweb-demo.uio.no/emreg/list

The response from EMREG contains a list in the following JSON format:

The list contains a parameter called “SingleFetch” saying whether a particular country has separate EMPs for each institution. In that case, after the country has been selected, the student will be presented with a list of all EMPs registered for that country, and for each EMP only the first institution on the list will be presented to the student (even though, for compatibility reasons, it will stil be delivered by EMREG as a list).

[Sending a request to the EMP](sending-a-request-to-the-emp.md)

[Receiving a response](receiving-a-response.md)

[Interpreting and handling the data](interpreting-and-handling-the-data.md)
