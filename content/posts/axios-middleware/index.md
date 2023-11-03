---
title: Configure axios to refresh access token
date: 2020-12-20
tags: [frontend]
# mermaid: true
# image: diagram.png
---

If you are working with an API which uses OAuth for authentication then you need to some way of refreshing your access token when it is expired to make sure a uninterrupted user experience. And for front-end application it is very important.
Axios is a promise based HTTP client which comes with a lot of great feature.
<!--more-->
 Interceptors is one of the great feature we are going to talk about. It gives you the ability to intercept requests or responses before they are handled by `then` or `catch` block. That means you can easily modify it before they reach their destination.

In your react app create a config file named `axiosConfig`.js and add these codes below.

You can go through the [documentation](https://github.com/axios/axios#interceptors) for further enquiry about **interceptors**.

```javascript
import axios from "axios";

const instance = axios.create({
	baseURL: `${process.env.REACT_APP_API_URL}`,
});

instance.interceptors.request.use(
	async (config) => {
		// get your token from your local storage or cookie
		const token = getToken();
		if (token) {
			// you can set your content-type or other parameter here
			config.headers["Authorization"] = "Bearer " + token;
		}
		return config;
	},
	(error) => {
		Promise.reject(error);
	}
);

instance.interceptors.response.use(
	(response) => {
		// Any status code that lie within the range of 2xx cause this function to trigger
		return response;
	},
	async function (error) {
		var originalRequest = error.config;

		if (error.response.status === 401 && !originalRequest._retry) {
			originalRequest._retry = true;

			const refreshToken = getRefreshToken();

			/**if only refresh token is removed somehow from your local storage
			 * remove auth token and return original request
			 * otherwise it will try to authenticate with the invalid authToken
			 * and will get invalid response each time which will be a fail attempt to login
			 * */
			if (refreshToken === undefined) {
				removeToken();
				return instance(originalRequest);
			}

			return instance
				.post(`/token/refresh`, { refreshToken })
				.then((response) => {
					setToken({
						authToken: response.data.token,
						refreshToken: response.data.refreshToken,
					});

					const newAuthToken = getToken();

					instance.defaults.headers.common["Authorization"] =
						"Bearer " + newAuthToken;

					originalRequest.headers["Authorization"] =
						"Bearer " + newAuthToken;

					return instance(originalRequest);
				})
				.catch((error) => {
					removeToken();
					redirectTo("/login");
					return Promise.reject(error);
				});
		}
		return Promise.reject(error);
	}
);

export default instance;
```

You can even further modify the code to your needs. If you need to add some additional headers or retry a request for other statuses you can do that.
