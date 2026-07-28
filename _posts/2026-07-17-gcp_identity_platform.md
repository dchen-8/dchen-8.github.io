---
layout: post
title: "Authentication with Google Cloud Platform(GCP) Identity Platform"
author: David Chen
email: david@davidc.dev
date: 2026-07-17
categories: document learnings writeup
---

* TOC
{:toc}

## Authentication with GCP's Identity Platform

Authentication is critical to any useful platform. There needs to be a way to login users so they can use the more advanced and tailored tools for each user rather than generic tools that could be useful.

The goal is to find a scalable solution that is able to integrate seamlessly into my current tech stack.

Some critical requirements is ease of use and ability for new users to choose email as a way to login but also have the flexibility for using social login like Google/Facebook.

## Systems considered:

### [Auth0](https://auth0.com/){:target="_blank"}

There were initially thought of that immediately dropped when I started looking at the pricing page. 

The free tier allowed up to 25.000 Monthly Active Users and then the first paid tier was 500 Monthly Active Users for $35 a month which made no sense to me. I don't see how going from a free plan to a paid plan can have such a high discrepancy. I was immediately turned away and did not bother trying the free plan. My concern would be sinking time into a standalone tool and hoping that it would work with my current tech stack and having to spend days trying to validate that claim. I spent a long time on Supabase trying to see if it could be my Postgres instance but based on all the time it took, had I just gone with CloudSQL from the beginning, I would not have wasted all that time. 

If I was doing this more leisurely I could have the time to check out new integrations/systems but if I am trying to build a business from the ground up, I need things to just work. 

### [Supabase](https://supabase.com/auth){:target="_blank"}

Supabase was definitely considered but had the same concern when I was trying to validate it for being used as a Postgres backend. 

I saw the pricing plan and thought that if I spent any time on Supabase, if I had to upgrade the lowest tier would be $25 a month which made little sense for me to try it. Trying this tool that says it'll make things easier but you get hit later with a bunch of issues do not sound like a good time to me.

I decided to possibly circle back to Supabase later if Google Cloud Platform - Identity Platform did not perform as expected.

### [Google Cloud Platform - Identity Platform](https://cloud.google.com/security/products/identity-platform?hl=en){:target="_blank"}

It was seamless to try. I turned on the API and then was able to easily turn on which kind of authentication I would want my front end to be able to support. I choose to do Email and Google. 

Next was the seamless integration into my React website where with some minor configurations, I was able to setup a Login/Register new user page through the use of AI. 

It just seemed to work really well. I was concerned about custom claims if I wanted to set up new users as an Editor/Admin/etc.

### [Firebase Authentication](https://firebase.google.com/docs/auth){:target="_blank"}

I poked around firebase and was surprised to learn that Identity Platform is directly connected to firebase auth. Firebase auth is built on top of Identity platform. Identity platform has a few more distinct features like OIDC and SAML support for enterprise login but they are very similar under the hood with both of them using the firebase auth and firebase admin package for python.

## Integrations:

These are some things that I thought about when deciding which system to utilize.

### Authentication vs Authorization

I knew how to authenticate via Firebase but I was not sure how I was able to authorize what each person has access to. I mulled it over and tried to see if I could implement Google Groups Integration or even a custom claim on each user when they sign up.

#### Google Groups

Was not possible for integration as there was not really a guaranteed google user that could be added into a google group to provide which group has access. I considered trying to roll my own access server type thing where I would create flat files and try to validate whether or not the user should have access but decided against it as it seemed like overkill for a small problem.

#### Custom Claims - Editor, Admin, Etc

I looked into add a custom claim for each user as it was created during registration but ran into problems when trying to deploy a cloud function v1 for the BeforeCreate function for the identity platform. Identity platform only accepts v1 cloud functions and it seemed very tedious to try to figure out the right configuration in how to assign a custom claim for each user.

Custom claims is like syntactic sugar on the claim, so along with some other specific fields, it can be added per auth through firebase and be stored directly and not in any other system. This would make it super handy to store key information that might be always needed during authentication. I was thinking I could add authorization rights baked right into the claim which could not be faked during validation.

I decided to make the assumption that a user that was logged in (which can be validated on the front end) will be able to do any kind of interaction while admin users will be possibly hardcoded or thrown into a database to be looked up later if this project expands.

#### User Profile Storage Location

I considered where if I wanted to have user settings being stored somewhere super accessible so that user settings would be lightning fast to access. Firebase has a realtime database which is like one giant dict which should be great to use to store user profile information. 

### JSON Web Tokens(JWT)

I knew about JWT but didn't know how to implement it. When I was trying to confirm how firebase does the authentication, I found out that it used JWT which is super nice. Makes things really easy and I can easily implement a validation framework on the backend to ensure that each route that needs authentication can check if the Bearer token is valid. 

Luckily the JWT token is set to timeout after an hour so that consistently stealing it should not be a risk. Though if they steal the token and refresh token, that could be a problem. The refresh might only work on my site so that should be more security. The refresh/auth are tied to my domain so it would fail if they are being refreshed from a different domain/source.

## Scaling

I wanted to make sure I was able to easily scale to as many monthly active users as possible without hitting any limited. It does seem like Identity Platform gives up to 50,000 MAU for free and then only starts charging a negligible amount per user after that.

## Conclusion

Spinning up an login process seemed daunting at first but going through the GCP's Identity Platform was pretty seamless after a bit of experimentation. I feel this might be the trend going forward, to fallback to enterprise level tools rather than try to shoehorn a fancy new thing that might not work right and have cut corners where the enterprise thing is big and beefy and just works.

## Next steps

I will keep an eye out for access issues when I start working on the Admin page where I should be able to do Tag management, Group Management and generally just be able to clean up my data as it gets displayed. 

I want to be able to change certain things but I also want the data to continue to flow regularly. 