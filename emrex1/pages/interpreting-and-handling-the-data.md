Interpreting and handling the data
==================================

[Technical guide](technical-guide.md)

The ELMO XML format contains the results themselves. The document is signed, using the XML DSig format, with the private key of the EMP that issued it. The public key can be obtained from EWP Registry. If signature verification fails, it means the results have been tampered with and MUST be rejected.

In addition to verifying the signature, the receiving client must ensure the results belong to the same person requesting them. ELMO includes the person’s name and birthday which can be used for this purpose. Both signature and person verification are provided by the EMC.

[Choosing the EMP](choosing-the-emp.md)

[Sending a request to the EMP](sending-a-request-to-the-emp.md)

[Receiving a response](receiving-a-response.md)