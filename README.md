# The transcoding logic of grpc-httpjson-transcoding

![flow chart](https://github.com/w12379564/transcoding/blob/master/transcoding.jpg)

1. Implement the Transcoder interface defined in [transcoder.h](https://github.com/grpc-ecosystem/grpc-httpjson-transcoding/blob/master/src/include/grpc_transcoding/transcoder.h). We can refer to the implementation in [Extensible Service Proxy](https://github.com/cloudendpoints/esp/blob/master/src/grpc/transcoding/transcoder_factory.cc#L61).

2. Initialize Transcoder factory for a specific service config. Holds the preprocessed service config and creates a Transcoder per each client request using the following information:
   - method call information
     - RPC method info
       - request & response message types
       - request & response streaming flags
     - HTTP body field path
     - request variable bindings
       - Values for certain fields to be injected into the request message.
   - request input stream for the request JSON coming from the client,
   - response input stream for the response proto coming from the backend.
  
   EXAMPLE:
   ```cpp
     TranscoderFactory factory(service_config);

     ::google::protobuf::io::ZeroCopyInputStream *request_downstream =
        CreateClientJsonRequestStream();

     ::google::protobuf::io::ZeroCopyInputStream *response_upstream =
        CreateBackendProtoResponseStream();

     unique_ptr<Transcoder> transcoder;
      // Creates a Transcoder object to transcode a single client request
      // call_info - contains all the necessary info for setting up transcoding
      // request_input - ZeroCopyInputStream that carries the JSON request coming
      //                 from the client
      // response_input - ZeroCopyInputStream that carries the proto response
      //                  coming from the backend
      // transcoder - the output Transcoder object
     status = factory.Create(request_info,
                             request_downstream,
                             response_upstream,
                             &transcoder);
   ```
   
3. Use the transcoder to translate the request and respone.

   EXAMPLE:
   ```cpp
    Transcoder* t = transcoder_factory->Create(...);

    const void* buffer = nullptr;
    int size = 0;
    while (backend can accept request) {
      if (!t->RequestOutput()->Next(&buffer, &size)) {
        // end of input or error
        if (t->RequestStatus().ok()) {
          // half-close the request
        } else {
          // error
        }
      } else if (size == 0) {
        // no transcoded request data available at this point; wait for more
        // request data to arrive and run this loop again later.
        break;
      } else {
        // send the buffer to the backend
        ...
      }
    }

    const void* buffer = nullptr;
    int size = 0;
    while (client can accept response) {
      if (!t->ResponseOutput()->Next(&buffer, &size)) {
        // end of input or error
        if (t->ResponseStatus().ok()) {
          // close the request
        } else {
          // error
        }
      } else if (size == 0) {
        // no transcoded response data available at this point; wait for more
        // response data to arrive and run this loop again later.
        break;
      } else {
        // send the buffer to the client
        ...
      }
    }
   ```
