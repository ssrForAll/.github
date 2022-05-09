### Before Reading
The name is not set and I am using sfa as a place holder acronym. Meaing **S**erver Side Rendering **F**or **A**ll

#  Purpose
Bring Server Side Rendering with Componenet based web development into the world of non node.js based langauges such as Rust and Kotlin.

## The Current Problem
If you try to build a modern web app you are pushed to use frameworks such as Vue and React that do support producing Single Site Pages that can be served by backend. However, it removes the ability to give dynamic(Data pulled from the database) meta data and requires more and more calls to the backend

## The Solution - bring Server Side Rendering the backend

A framework similar to Vue however, support backend rendering for partial in the backend.


## Routing
Routing would use a mix of frontend handling and backend handling. Initial requests would be handled by default by the backend however, once in the frontend the developer can use different link types to force a full page load or just a partial page load.

### Examples

The frontend code would look something like this
#### base.sfa
```vue
<template>
  <defaultMeta>
    <!-- Sets Default meta tags for all pages-->
  </defaultMeta>
  <NavBar c-render />
</template>


<script lang="ts">
import { defineComponent } from "sfa";
import NavBar from "~/components/nav/NavBar.sfa";
export default defineComponent({
  components: { NavBar },
  setup() {
    console.log("Hey");
  },
});
</script>
```

#### admin.sfa
```vue


// `include-parent` is equal to true by default. But I am doing this for example. It means that it will include the parent elements
// The parent element designed for all is found in base.sfa. you can also set a custom parent via parent="admin_parent.sfa"

<template include-parent=true>
  <div class="w-full">
    <!-- All of the b-if statements are only included if they meet the required permission.  The components are pulled and included in the response.  -->
    <SubNavBar v-model="page">
      <LinkNavItem b-if="user.permissions.admin || user.permissions.user_manager" href="/admin/users" icon="user" name="Users" />
      <LinkNavItem b-if="user.permissions.admin || user.permissions.repository_manager" href="/admin/storages" icon="box" name="Storages" />
    </SubNavBar>
        <!-- All of the b-if statements are only included if they meet the required permission.  The components are pulled and included in the response.  -->
        <!-- The f-if will only render if the frontend condition is met  -->

    <Storages b-if="user.permissions.admin || user.permissions.user_manager" class="mt-2 md:mt-0" f-if="page == 'storages'" />
    <Users b-if="user.permissions.admin || user.permissions.repository_manager" class="mt-2 md:mt-0" f-if="page == 'users'" />
  </div>
</template>

<script lang="ts">
import Storages from "@/components/Storages.sfa";
import Users from "@/components/Users.sfa";
import { routeUtil } from "sfa-router";
import SubNavBar from "@/components/common/nav/SubNavBar.sfa";
import LinkNavItem from "../../components/common/nav/LinkNavItem.sfa";
export default defineComponent({
  components: {
    Storages,
    Repositories,
    Users,
    UpdateUser,
    Me,
    SubNavBar,
    LinkNavItem,
  },
  initialize(){
   // Gets current route. Split into an array. Gets the 2nd index it will equal a string or undefined
   let page =  routeUtil.getCurrentPage()[1];
   return {page};
  }
});
</script>
```
### repository.sfa
```vue
<template>
  <meta>
    <!-- Overrides default meta for custom tags-->
    <title>{{repository.name}}</title>'
    <meta property="og:description" content="Repository {{repository.name}}" />
  </meta>
  <div class="grid grid-row-2 gap-4">
    <div class=" m-2 ">
      <RepositoryBadge :repository="repository" />
    </div>
    <div class=" m-2">
      <MavenRepoInfo
        v-if="repositoryType === 'Maven'"
        :repository="repository"
      />
    </div>
  </div>
</template>
<style scoped></style>
<script lang="ts">
import MavenRepoInfo from "@/components/repo/types/maven/MavenRepoInfo.vue";
import { defineComponent} from "sfa";

import RepositoryBadge from "@/components/badge/RepositoryBadge.vue";
export default defineComponent({
  components: { MavenRepoInfo, RepositoryBadge },
});
</script>
```

This route would be assigned for `/admin` or `/admin/{file:.*}` meaning it can take a route such as /admin or `/admin/ANYTHING_ELSE/WITH_SLASHES/TOO`

```rust
pub async fn admin(pool: web::Data<DbPool>,sfa: web::Data<SFA> r: HttpRequest) -> Result<SFAResponse, InternalError> {
  // Include your code for getting the user from the HttpRequest
  let user = None;
  if user.is_none() {
    // push would use trigger a redirect response. However instead of a HTTP response with the redirect. it sends a HTTP response of that page. Plus a special command to change the route in the browser. Giving a more effiecient response
    sfa.push(format!("login?r={r.path()}"))
  }else {
  // sfa_context! would be a macro similar to serde_json::json! just building the data that would be passed into the template
    sfa.render(sfa_context!({user: user}, "admin.sfa")
  }
}
#[get("/repository/{storage}/{repository}")]
pub async fn repository(pool: web::Data<DbPool>,sfa: web::Data<SFA>, path: web::Path<(String, String>, r: HttpRequest) -> Result<SFAResponse, InternalError> {
  // Include your code for getting the user from the HttpRequest
  let user = None;
  // Include the code to Repository
  let repository = None;
  if repository.is_none() {
    // Sends a frontend response for 404. Using the 404 page designed
    sfa.error(404);
  }else {
  // sfa_context! would be a macro similar to serde_json::json! just building the data that would be passed into the template
   // Also user will become undefined in the client
    sfa.render(sfa_context!({user: user, repository: repository}, "admin.sfa")
  }
}
```


All conditional statements found in frameworks in Vue such as v-if v-for ... Would have two versions s-if c-if  s-for c-for
b-{statement_type} would mean it has to be rendered via the backend. c-{statement_type} would mean it is renered via client.

You can mark specific elements to render in the frontend or backend via c-render or s-render. However, SFA will try to use an optimal or smart route by default. Looking via context based on the uses of c-if or c-for. Or if a variable gets defined in the client. However, it can be forced need by.

In my backend example. I will use Actix however, it would support any web framework

So lets say those SubNavBar will do a f-push="/admin/users" and f-push="/admin/storages" meaning they will change the route but no trigger the backend to re send the page

Bugs that will occur f-push="/repositories/public/maven" in a case like this. it changes outside of the realm of /admin and that page requires a full re render including other data such as the Repository Information. This will result in a 404 or something and a error message indicating that this url is not found on the current frontend context.

## Passing Data
When data is sent into the page render in the backend. By default it would use this data inside the backend render. Then it will put the data as a global variable in the web scripts. You can mark the data to just use backend render then not pass it the frontend for extra rendering

In our example above NavBar checks to see if user is defined and if so it says the user is logged in.

### Macro Explained
sfa_context!({user: user} The user object would go through the stages of rendering on the backend. And then in a script tag injected on the page would create const user = {JSON version of the user object}.

You can do sfa_context!({user: {data: user, client-render: false}}) and mark it to not go into the client render
# Targetted Languages
In my initial design we will target a Rust backend.

However, I will be making a JVM version(Probably written in Kotlin) and then later a Go version of the backend renderer. However, the compiler for the frontend would still be in Rust
