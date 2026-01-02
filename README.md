# Feature Implementation: User Role Gating & Selection

## Context: We are implementing a "Gate" (InternGatingRound) that intercepts new users after login. They must select a role (Engineer, Doctor, etc.) which is then saved to the database. Once saved, they bypass this gate in future logins.



## 1. Database Change

Goal: Create a place to store the role.

Open H2 Console (or your DB client).

Run this exact SQL command:

```
ALTER TABLE login ADD COLUMN userRole VARCHAR(255) DEFAULT '';
```

## 2. Java Backend Changes

### A. File: src/main/java/com/thb/login/RegisterUser.java
Action: Add method signatures to the Interface.

Open 
RegisterUser.java

Search for:
```
public String register(String username ...
```

Below that line, add these two lines:

```
public String getUserRole(String username) throws SQLException;
public void setUserRole(String username, String role) throws SQLException;
```

### B. File: 

src/main/java/com/thb/login/impl/RegisterUserImpl.java
Action: Update SQL and implement logic.

Step 0: Check Imports Ensure these imports are present at the top of the file:

import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import com.thb.db.DatabaseConnection;
Step 1: Fix the Insert Query

Search for the register method.

Inside it, find the line starting with String sql = "insert into ...
Replace that entire line with:
```
String sql = "insert into " + prefixForTable + "login values( ? , ? , ? , ?, ? ) ";
```
(Note the 5th question mark)

Scroll down slightly to where the parameters are set (preparedStatement.setString...).
After preparedStatement.setString(4, urlData);, insert this new line:
```
preparedStatement.setString(5, ""); // Set default empty role
```

Step 2: Add New Methods

Go to the very bottom of the file.
Find the last closing brace } of the class.
Just before that last brace, paste this code:

```

    public String getUserRole(String username) throws SQLException {
        String role = "undefined";
        Connection connection = DatabaseConnection.getConnection();
        try {
            String sql = "select userRole from " + prefixForTable + "login where username = ?";
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, username);
            ResultSet res = preparedStatement.executeQuery();
            if (res.next()) {
                String dbRole = res.getString(1);
                if (dbRole != null && !dbRole.trim().isEmpty()) {
                    role = dbRole;
                }
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
        return role;
    }
    public void setUserRole(String username, String role) throws SQLException {
        Connection connection = DatabaseConnection.getConnection();
        try {
            String sql = "update " + prefixForTable + "login set userRole = ? where username = ?";
            PreparedStatement preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, role);
            preparedStatement.setString(2, username);
            preparedStatement.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            connection.close();
        }
    }

```

### C. File: src/main/java/com/thb/webservice/rest/RegisterUserService.java
Action: Create API endpoints.

Step 0: Check Imports Ensure these imports are present at the top of the file:

import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.core.Response;
import java.sql.SQLException;
Step 1: Add Endpoints

Open 
RegisterUserService.java
.
Search for @Component.
Inside the class, finding existing methods like @POST.
After the last method (but before the final closing brace }), paste these two methods:

```
@GET
    @Path("getUserRole/{username}")
    public Response getUserRole(@PathParam("username") String username) {
        String role = "undefined";
        try {
            role = registerUser.getUserRole(username);
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return Response.status(200).entity(role).build();
    }
    @GET
    @Path("setUserRole/{username}/{role}")
    public Response setUserRole(@PathParam("username") String username, @PathParam("role") String role) {
        try {
            registerUser.setUserRole(username, role);
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return Response.status(200).entity("Role Saved").build();
    }
```

## 3. Frontend Changes

### A. New File: src/main/webapp/InternGatingRound.html
Action: Create this file. This is the UI for choosing a role.

Create a new file named 
InternGatingRound.html in src/main/webapp.

Paste the entire content below into it:

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Select Your Role</title>
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.5.1/jquery.min.js"></script>
    <style>
        body { background-color: #0B0C10; color: white; display: flex; justify-content: center; align-items: center; height: 100vh; font-family: sans-serif; }
        .role-cards { display: flex; gap: 20px; flex-wrap: wrap; justify-content: center; }
        .role-card { background: #1F2833; padding: 20px; border: 2px solid #45A29E; border-radius: 10px; cursor: pointer; min-width: 100px; text-align: center; }
        .role-card:hover { background: #45A29E; color: black; }
        #role-selection-container { display: none; text-align: center; }
    </style>
</head>
<body>
    <div id="loading-message">Verifying User Profile...</div>
    <div id="role-selection-container">
        <h1>Who are you?</h1>
        <div class="role-cards">
            <div class="role-card" onclick="selectRole('Engineer')">Engineer</div>
            <div class="role-card" onclick="selectRole('Doctor')">Doctor</div>
            <div class="role-card" onclick="selectRole('Artist')">Artist</div>
            <div class="role-card" onclick="selectRole('Gamer')">Gamer</div>
            <div class="role-card" onclick="selectRole('Writer')">Writer</div>
            <div class="role-card" onclick="selectRole('Student')">Student</div>
        </div>
    </div>
    <script>
        // --- LOGIC START ---
        var apiBaseUrl = "/portal/portal/register"; 
        function getParam(name) {
            name = name.replace(/[\[\]]/g, '\\$&');
            return new URL(window.location.href).searchParams.get(name);
        }
        
        // Extract username from obfuscated param
        var params = new URLSearchParams(window.location.search);
        var currentUsername = null;
        for (const [key, value] of params.entries()) {
            if (key.includes("ygdbrw")) {
                currentUsername = value.includes("@") ? value.split("@")[0] : value;
                break;
            }
        }
        var originalQueryParams = window.location.search;
        $(document).ready(function () {
            if (!currentUsername) {
                // Fallback if no user found
                window.location.replace("ProfileLayoutDivPercentage.html" + originalQueryParams);
                return;
            }
            
            // Check if role exists
            $.ajax({
                url: apiBaseUrl + "/getUserRole/" + currentUsername,
                type: 'GET',
                success: function (role) {
                    if (role && role !== "undefined" && role.trim() !== "") {
                        // Role found! Go to profile.
                        window.location.replace("ProfileLayoutDivPercentage.html" + originalQueryParams);
                    } 
                    else {
                        // Role missing? Show selection screen.
                        $("#loading-message").hide();
                        $("#role-selection-container").fadeIn();
                    }
                },
                error: function() {
                     // Error? Show screen just in case.
                     $("#loading-message").hide();
                     $("#role-selection-container").fadeIn();
                }
            });
        });
        function selectRole(role) {
            $("#role-selection-container").hide();
            $("#loading-message").text("Saving Profile...").show();
            $.ajax({
                url: apiBaseUrl + "/setUserRole/" + currentUsername + "/" + role,
                type: 'GET',
                success: function () {
                    window.location.replace("ProfileLayoutDivPercentage.html" + originalQueryParams);
                },
                error: function() {
                    alert("Error saving role. Try again.");
                    $("#loading-message").hide();
                    $("#role-selection-container").show();
                }
            });
        }
    </script>
</body>
</html>

```

### B. File: src/main/webapp/UserLogin.html

Action: Update Redirects.

Open UserLogin.html.

Search/Find every occurrence of ProfileLayoutDivPercentage.html.

You will find it in window.location.replace(...) calls.

Replace ProfileLayoutDivPercentage.html with InternGatingRound.html in those lines.

Example Change:

FROM: window.location.replace(SERVER + "/ProfileLayoutDivPercentage.html?ygdbr" ...

TO: window.location.replace(SERVER + "/InternGatingRound.html?ygdbr" ...

## 4. Verification

- Stop Server (if running).
- Project -> Clean... in Eclipse (Critical step!).
- Start Server.
- Open Browser -> Login with a new user.
- Outcome: "Who are you?" screen appears.
- Click "Engineer".
- Outcome: Redirects to Profile.
- Logout & Login again with same user.
- Outcome: Goes straight to Profile (skips selection).
- Troubleshooting
- Error related to SQLException or List or Response?
- Check Step 0 in Java sections. You likely missed adding the Imports to the top of the file.
- 404 Not Found when clicking "Engineer"?
- You probably didn't restart variables/server correctly. Restart the server.
- Check InternGatingRound.html -> apiBaseUrl. It should be /portal/portal/register.