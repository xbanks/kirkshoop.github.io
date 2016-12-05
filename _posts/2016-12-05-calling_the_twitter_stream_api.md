---
title: Calling the twitter stream API
layout: post
date: 2016-12-05
---

[![twitter for president-elect](/assets/twitter_analysis_president_elect.gif)](https://www.youtube.com/watch?v=QFcy-jQpvBg)
[youtube](https://www.youtube.com/watch?v=QFcy-jQpvBg)

## twitter analysis application
The code for the application is on [github](https://github.com/kirkshoop/twitter). I deveop and test this application on OS X, but I chose all the dependencies to make it likely to work on other platfroms with minimum effort. On OS X I was able to `brew install` some dependencies and use `CMake` to install others.

## twitter stream API
I found this [c++ gist that calls the twitter stream api](https://gist.github.com/komasaru/9c78a278f6916548f146).

To use it I had to go to the twitter site and create an app.

[![twitter applications](/assets/twitter_apps.png)](https://apps.twitter.com/)

Once created, the twitter application will have:

 - Consumer Key (API Key)
 - Consumer Secret (API Secret)
 - Access Token
 - Access Token Secret

These are required to use oauth to sign the URL for the stream api call. I made these commandline arguments for the twitter analysis application.

## twitterrequest
This function signs the url and calls the twitter stream api.

The keys are used to call `oauth_sign_url2`, which returns the signed url.

```cpp

            signedurl = oauth_sign_url2(
                URL.c_str(), NULL, OA_HMAC, method.c_str(),
                CONS_KEY.c_str(), CONS_SEC.c_str(), ATOK_KEY.c_str(), ATOK_SEC.c_str()
            );

```

The signed url and method are used to create an http request. The result will emit the strings from the body as they arrive. The request will not start until `subscribe()` is called on the request.

```cpp

        return factory.create(http_request{url, method}) |
            rxo::map([](http_response r){
                return r.body.chunks;
            }) |

```

Emiting the strings as they arrive is a requirement for the http library used to call the twitter stream api. The stream api returns `\r\n` delimited json documents, but never completes.

__the complete function__

```cpp

auto twitterrequest = [](observe_on_one_worker tweetthread, ::rxcurl::rxcurl factory, string URL, string method, string CONS_KEY, string CONS_SEC, string ATOK_KEY, string ATOK_SEC){
    return observable<>::defer([=](){
        string url;
        {
            char* signedurl = nullptr;
            RXCPP_UNWIND_AUTO([&](){
                if (!!signedurl) {
                    free(signedurl);
                }
            });
            signedurl = oauth_sign_url2(
                URL.c_str(), NULL, OA_HMAC, method.c_str(),
                CONS_KEY.c_str(), CONS_SEC.c_str(), ATOK_KEY.c_str(), ATOK_SEC.c_str()
            );
            url = signedurl;
        }

        return factory.create(http_request{url, method}) |
            rxo::map([](http_response r){
                return r.body.chunks;
            }) |
            merge(tweetthread);
    }) |
    twitter_stream_reconnection(tweetthread);
};

```

The `defer` ensures that a completely new http request is made each time `subscribe()` is called on the result.

At one point, there there was a bug where the same signed url was being reused for reconnects. Putting the signing into the defer fixed this, since each new request got a newly signed url.

## twitter_stream_reconnection

This function implements the [twitter stream etiquette](https://dev.twitter.com/streaming/overview/connecting). 

Create a timeout error if nothing has arrived for 90 seconds. The twitter stream api is supposed to send at least an empty line every 30 seconds.

```cpp

    timeout(seconds(90), tweetthread) |

```

Timeout errors should reconnect immediately.

> `empty<string>()` completes immediately. When the stream completes `repeat()` re-subscribes to the `defer` above, which starts a new request.

```cpp

    catch (const timeout_error& ex) {
        cerr << "reconnecting after timeout" << endl;
        return observable<>::empty<string>();
    }

```

After a TCP error, reconnect immediately. 

```cpp

    case errorcodeclass::TcpRetry:
        cerr << "reconnecting after TCP error" << endl;
        return observable<>::empty<string>();

```

After a recoverable http error, wait 5 seconds and then reconnect.

> `timer()` emits a long, but this is a stream of `string`. `stringandignore()` converts to the correct type.

```cpp

    case errorcodeclass::ErrorRetry:
        cerr << "error code (" << ex.code() << ") - ";
    case errorcodeclass::StatusRetry:
        cerr << "http status (" << ex.httpStatus() << ") - waiting to retry.." << endl;
        return observable<>::timer(seconds(5), tweetthread) | stringandignore();

```

When the server is limiting our access, wait 1 minute and then reconnect.

```cpp

    case errorcodeclass::RateLimited:
        cerr << "rate limited - waiting to retry.." << endl;
        return observable<>::timer(minutes(1), tweetthread) | stringandignore();

```

Unrecoverable errors do not reconnect.

```cpp

    return observable<>::error<string>(ep, tweetthread);

```

__the complete function__

```cpp

auto twitter_stream_reconnection = [](observe_on_one_worker tweetthread){
    return [=](observable<string> chunks){
        return chunks |
            // https://dev.twitter.com/streaming/overview/connecting
            timeout(seconds(90), tweetthread) |
            on_error_resume_next([=](std::exception_ptr ep) -> observable<string> {
                try {rethrow_exception(ep);}
                catch (const http_exception& ex) {
                    cerr << ex.what() << endl;
                    switch(errorclassfrom(ex)) {
                        case errorcodeclass::TcpRetry:
                            cerr << "reconnecting after TCP error" << endl;
                            return observable<>::empty<string>();
                        case errorcodeclass::ErrorRetry:
                            cerr << "error code (" << ex.code() << ") - ";
                        case errorcodeclass::StatusRetry:
                            cerr << "http status (" << ex.httpStatus() << ") - waiting to retry.." << endl;
                            return observable<>::timer(seconds(5), tweetthread) | stringandignore();
                        case errorcodeclass::RateLimited:
                            cerr << "rate limited - waiting to retry.." << endl;
                            return observable<>::timer(minutes(1), tweetthread) | stringandignore();
                        case errorcodeclass::Invalid:
                            cerr << "invalid request - exit" << endl;
                        default:
                            return observable<>::error<string>(ep, tweetthread);
                    };
                }
                catch (const timeout_error& ex) {
                    cerr << "reconnecting after timeout" << endl;
                    return observable<>::empty<string>();
                }
                catch (const exception& ex) {
                    cerr << ex.what() << endl;
                    terminate();
                }
                catch (...) {
                    cerr << "unknown exception - not derived from std::exception" << endl;
                    terminate();
                }
                return observable<>::error<string>(ep, tweetthread);
            }) |
            repeat(0);
    };
};

```

With this we get a stream of strings that contain `\r\n` delimited json documents.

## up next
parsing json documents
