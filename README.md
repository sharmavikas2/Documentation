
# Github-Repo-Creator
A backstage plugin that allow users to creator repositories under an orgnistation. 
## Technologies Used
- [Backstage by Spotify](https://backstage.io/)
- [Github Api](https://docs.github.com/en/rest?apiVersion=2022-11-28)
## Functionalities
- Create repositories under an organisation.
## How to set up the project
#### 1. Create a backstage project using the following command.
```
npx @backstage/create-app@latest
```
```
Enter y to proceed
```
```
Enter a name for your app and hit enter
```
You will see an interface like this

![Backstage project creation interface](https://backstage.io/assets/images/create-app-output-a47b1569049245f0993e1b6fa07589dd.png) 

The folder structure will be like this
```
app
├── app-config.yaml
├── catalog-info.yaml
├── package.json
└── packages
    ├── app
    └── backend
```
#### 2. Creating a plugin
To create a frontend plugin run the following command (make sure you are inside your backstage project direactory)
```
yarn new --select plugin
```
```
Enter a name for your plugin (github-repo-creator) and hit enter
```
You will see an interface like this

![Plugin creation interface](https://backstage.io/assets/images/create-plugin_output-9d6c4192d058ed85c75109a4d907eae6.png)

Now your directory structure will look like this
```
app
├── app-config.yaml
├── catalog-info.yaml
├── package.json
├── examples
|   └── org.yaml
├── packages
|    ├── app
|    └── backend
└── plugins
     └── github-repo-creator
```

### 3. Setting up authentication
- Firstly we have setup our app inside the github. 
```
Go to Github -> Settings -> Developer Settings -> Register a new application
```
```
Enter a name for your app
Homepage URL: http://localhost:3000
Authorization callback URL: http://localhost:7007/api/auth/github/handler/frame
```
Your final form will look like this
![App Registration Interface](https://firebasestorage.googleapis.com/v0/b/file-sharing-cf3f8.appspot.com/o/A4HQYaz8wqdmrJMDsY2lW7aWRwY2%2FScreenshot%202024-06-22%20101427.png?alt=media&token=9899cc70-979d-4237-acc9-b645ef81f5a4)

Click register applicaiton

```
Copy the client Id
Click on generate client secret and copy it
```
- Create an Organisation and then authorize your app to access the organisation
```
Go to organisation -> your organisation -> settings -> Third party access -> oauth applicaiton policy
```
Grant access to your newly created app so that it can access the organisation.
- Setting up authentication inside the backstage app
Go to app-config.yaml
```
app
├── app-config.yaml
├── catalog-info.yaml
├── package.json
├── examples
|   └── org.yaml
├── packages
|    ├── app
|    └── backend
└── plugins
     └── github-repo-creator
```

Inside the app-config.yaml, under the auth section inside providers we have to add github. Paste the following code inside provider
```
github:
      development:
        clientId: your_client_Id
        clientSecret: your_client_secret
        signIn:
          resolvers:
            # Only one of these
            - resolver: usernameMatchingUserEntityName
```
Replace content of clientId and client secret with the client id and client secret obtained after registering app on github.

Now, we have to add the github authentication plugin to our backend to allow our backend to authenticate user using github
```
Go to packages -> backend -> src -> index.ts
```
```
packages
├───app
│   ├───public
│   └───src
│       └───components
│           ├───catalog
│           ├───Root
│           └───search
└───backend
    └───src
        └───index.ts
```
Import this package
```
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```
Now, we have to store the username of the user inside the org.yaml so that resolver can know with what name it  has to match the name of user logging in to avoid the unauthorised user to login.

```
Go to examples -> org.yaml 
```
```
examples
   └── org.yaml
```
Inside the org.yaml in the apiVersion with ``` kind : User ``` replace the name inside metadata with github username of user
```
apiVersion: backstage.io/v1alpha1
kind: User
metadata:
  name: github_username
spec:
  memberOf: [guests]
```

Now, we have setup the backend we have to add a signup page to authenticate  the user.
```
Go to packages -> app -> src -> App.tsx
```
Above the app function
```
const app = createApp({
 // existing code
 components: {
    SignInPage: props => (
      <SignInPage
        {...props}
        providers={[
          'guest',
        ]}
      />
    ),
  },
});
```
Create an object named githubAuthCfg it will consist of all the things required to set up the sign up page for user using github.
```
import { GithubRepoCreatorPage } from '@internal/backstage-plugin-github-repo-creator';

const githubAuthCfg = {
  id: 'github-auth-provider',
  title: 'GitHub',
  message: 'Sign in using GitHub',
  apiRef: githubAuthApiRef,
};
```

Replace the existing components code inside the app with this 
```
const app = createApp({
  //existing code
  components: {
    SignInPage: props => (
      <SignInPage {...props} auto providers={[githubAuthCfg]} />
    ),
  },
});

```
This completes our authentication set up

4. #### Setting up our plugin
```
Go to plugins -> github-repo-creator -> src
```
Remove the existing component folder

```
Create a new component folder
Inside it create a MainComponent folder
Inide the MainComponent folder create two files index.ts and Main.tsx
```

Now inside the Main.tsx we will be creating the UI of our plugin. Copy the following code to create a UI of the plugin. Replace `your-org` with name of your org
```
import React, { useEffect, useState } from 'react';
import {
  Grid,
  Input,
  InputLabel,
  Card,
  CardContent,
  Button,
  CardHeader,
} from '@material-ui/core';
import {
  Header,
  Page,
  Content,
  ContentHeader,
  HeaderLabel,
  SupportButton,
} from '@backstage/core-components';
import { githubAuthApiRef, useApi } from '@backstage/core-plugin-api';
import axios from 'axios';
import { red } from '@material-ui/core/colors';

export const Main = () => {
  const githubAuthApi = useApi(githubAuthApiRef);
  const [repos, setRepos] = useState([{ id: 0, name: '' }]);
  const [username, setUsername] = useState('NO USER');
  const [accessToken, setAccessToken] = useState('access token');
  const [repoName, setRepoName] = useState('');
  const [repoDescription, setRepoDescription] = useState('');
  const [org, setOrg] = useState('');
  const [submitting, setSubmitting] = useState(false);
  const [message, setMessage] = useState('');
  const [url, setUrl] = useState('');
  const [hover, setHover] = useState(false);
  useEffect(() => {
    const fetchRepos = async () => {
      await githubAuthApi
        .getAccessToken(['repo'])
        .then(async (token: string) => {
          setAccessToken(token);
          await axios
            .get('https://api.github.com/user', {
              headers: { Authorization: `token ${token}` },
            })
            .then(async response => {
              setUsername(response.data.login);
              await axios
                .get(
                  `https://api.github.com/users/${response.data.login}/repos`,
                  {
                    headers: { Authorization: `token ${token}` },
                  },
                )
                .then(res => {
                  setRepos(res.data);
                });
            });
        });
    };
    fetchRepos();
  }, [githubAuthApi]);
  const submit = async () => {
    /* eslint no-console: ["error", { allow: ["warn", "error","log"] }] */
    if (repoName === '' || repoDescription === '') {
      setMessage('Please fill all the fields');
    } else {
      setSubmitting(true);
      await axios
        .post(
          `https://api.github.com/orgs/your-org/repos`,
          {
            name: repoName,
            description: repoDescription,
            private: false,
          },
          {
            headers: { Authorization: `token ${accessToken}` },
          },
        )
        .then(res => {
          console.log(res.data.html_url);
          setMessage('Repository created successfully at ');
          setUrl(res.data.html_url);
          setRepoName('');
          setRepoDescription('');
          setSubmitting(false);
        })
        .catch(err => {
          console.error(err);
          setMessage(err.code);
          setRepoName('');
          setRepoDescription('');
          setSubmitting(false);
        });
    }
  };
  return (
    <Page themeId="tool">
      <Header title="Welcome to github-repo-creator!">
        <HeaderLabel label="Owner" value="Team X" />
        <HeaderLabel label="Lifecycle" value="Alpha" />
      </Header>
      <Content>
        <ContentHeader title=" ">
          <SupportButton>
            A plugin that helps you create github repositories
          </SupportButton>
        </ContentHeader>
        <Grid container spacing={3} direction="column">
          <Grid
            item
            style={{
              width: '100%',
              display: 'flex',
              justifyContent: 'center',
              alignItems: 'center',
              padding: '20px',
              height: '60vh',
              fontFamily:
                'Helvetica Neue, Helvetica, Roboto, Arial, sans-serif',
            }}
          >
            <Card
              style={{
                width: '40%',
                display: 'flex',
                justifyContent: 'center',
                alignItems: 'center',
                padding: '10px 20px 0px',
              }}
            >
              <CardHeader title="Github Repository Creator" />
              <CardContent
                style={{
                  width: '100%',
                  margin: '0',
                }}
              >
                <InputLabel
                  style={{
                    marginBottom: '5px',
                    color: '#1e293b',
                  }}
                >
                  Repository Name<span style={{ color: 'red' }}>*</span>
                </InputLabel>
                <input
                  value={repoName}
                  disabled={submitting}
                  onChange={e => setRepoName(e.target.value)}
                  placeholder="Enter Repository Name"
                  style={{
                    width: '100%',
                    padding: '12px',
                    fontSize: '1rem',
                    borderRadius: '15px',
                    border: '1px solid black',
                    outline: 'none',
                    marginBottom: '1.25rem',
                    color: '#1e293b',
                    height: '2.5rem',
                    lineHeight: '1.5rem',
                    fontWeight: 500,
                  }}
                />
                <InputLabel style={{ marginBottom: '5px', color: '#313144' }}>
                  Repository Description<span style={{ color: 'red' }}>*</span>
                </InputLabel>
                <input
                  value={repoDescription}
                  disabled={submitting}
                  onChange={e => setRepoDescription(e.target.value)}
                  placeholder="Enter Repository Description"
                  style={{
                    width: '100%',
                    padding: '12px',
                    fontSize: '1rem',
                    borderRadius: '15px',
                    border: '1px solid black',
                    outline: 'none',
                    marginBottom: '1.25rem',
                    color: '#1e293b',
                    height: '2.5rem',
                    lineHeight: '1.5rem',
                    fontWeight: 500,
                  }}
                />
                <button
                  onClick={() => {
                    submit();
                  }}
                  onMouseEnter={() => setHover(true)}
                  onMouseLeave={() => setHover(false)}
                  disabled={submitting}
                  style={{
                    width: '100%',
                    display: 'flex',
                    fontSize: '1rem',
                    justifyContent: 'center',
                    alignItems: 'center',
                    marginTop: '0.5rem',
                    padding: '6px 12px',
                    color: hover ? 'white' : '#1e293b',
                    border: '1px solid black',
                    borderRadius: '15px',
                    background: hover ? 'black' : 'white',
                    lineHeight: '1.5rem',
                    fontWeight: 500,
                    cursor: 'pointer',
                  }}
                >
                  Create Repository
                </button>
              </CardContent>
              <label
                hidden={message === ''}
                style={{
                  width: '100%',
                  display: 'flex',
                  justifyContent: 'center',
                  alignItems: 'center',
                  padding: '10px 6px',
                  fontSize: '1rem',
                  color: '#1e293b',
                }}
              >
                {message}
              </label>
              <a
                hidden={url === ''}
                href={url}
                style={{
                  fontSize: '1rem',
                  color: 'navy',
                  width: '100%',
                  display: 'flex',
                  justifyContent: 'center',
                  alignItems: 'center',
                  padding: '0px 6px 15px',
                }}
              >
                {url}
              </a>
            </Card>
          </Grid>
        </Grid>
      </Content>
    </Page>
  );
};

```
Now let us breakdown the code 
```
const [accessToken, setAccessToken] = useState('access token')
useEffect(() => {
    const fetchRepos = async () => {
      await githubAuthApi
        .getAccessToken(['repo'])
        .then(async (token: string) => {
          setAccessToken(token);
          await axios
            .get('https://api.github.com/user', {
              headers: { Authorization: `token ${token}` },
            })
            .then(async response => {
              setUsername(response.data.login);
              await axios
                .get(
                  `https://api.github.com/users/${response.data.login}/repos`,
                  {
                    headers: { Authorization: `token ${token}` },
                  },
                )
                .then(res => {
                  setRepos(res.data);
                });
            });
        });
    };
    fetchRepos();
  }, [githubAuthApi]);
```
This part of code is responsible for extracting the access token so that it can be used to access the organisation on the behalf of the user.

``
githubAuthApiRef is a reference to the GitHub authentication API within Backstage. This API helps your application interact with GitHub, especially for tasks that require authentication, such as fetching repositories, creating repositories, or getting user details from GitHub.
``

``
useApi is a function provided by Backstage. It allows you to access different APIs (Application Programming Interfaces) that Backstage provides. APIs are like tools or services that you can use to do specific tasks, like communicating with GitHub in this case.
``

The token is extracted using the following code and is stored inside the state variable ``accessToken`` .
```
await githubAuthApi.getAccessToken(['repo']).then(async (token: string) => { setAccessToken(token);})
```
The getAccessToken helps us to extract the token.

The following function helps us in creating the repository in an orgnistation.

```
 const submit = async () => {
    /* eslint no-console: ["error", { allow: ["warn", "error","log"] }] */
    if (repoName === '' || repoDescription === '') {
      setMessage('Please fill all the fields');
    } else {
      setSubmitting(true);
      await axios
        .post(
          `https://api.github.com/orgs/your-org/repos`,
          {
            name: repoName,
            description: repoDescription,
            private: false,
          },
          {
            headers: { Authorization: `token ${accessToken}` },
          },
        )
        .then(res => {
          console.log(res.data.html_url);
          setMessage('Repository created successfully at ');
          setUrl(res.data.html_url);
          setRepoName('');
          setRepoDescription('');
          setSubmitting(false);
        })
        .catch(err => {
          console.error(err);
          setMessage(err.code);
          setRepoName('');
          setRepoDescription('');
          setSubmitting(false);
        });
    }
  };
```

This code will create a respository under your orgs. It is sending a post request to `https://api.github.com/orgs/your-org/repos` with an object containing information regarding the repo. If all goes well the respository will be created under organization and will display the url of the repo on the ui of the plugin.

Now go to index.ts.It is  meant for importing the our MainComponent.

``
export { Main } from './Main';
``

Now we to import our component inside the plugin.ts it is responsible for displaying our component inside it.
```
export const GithubRepoCreatorPage = githubRepoCreatorPlugin.provide(
  createRoutableExtension({
    name: 'GithubRepoCreatorPage',
    component: () => import('./components/MainComponent').then(m => m.Main),
    mountPoint: rootRouteRef,
  }),
);
```
This function create a new page for inside our  githubRepoCreatorPlugin and will display our Main component.

We are now run with set up.

## How to use the plugin
- Firstly run the backstage project using 
```
yarn dev
```
- Now it will redirect you to ``http://localhost:3000/``
- Sign Up with github make sure you are signning with same github account whose name is inside the org.yaml.
- On successfully signnig in, go to  ``http://localhost:3000/github-repo-creator`` to access the plugin.
- Enter a name and description for repository and hit create repository. On success, it will display a link to repository. Otherwise it will show the error message.
