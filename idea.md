Idea:

1. Run everything on https on local ie nginx to use https for localhost, This will allow us to test preview server since gtm allows only https urls for preview server.
2. Run preview mode on different port and map that for route /gtm/debug using nginx, This way we can set both server and preview server under same url.
3. Test all scenarios, add, edit etc.
4. Automate all process for different tenants and tenant names.
5. test for custom domain mapping
6. Final testing and production.

Next phase:

- Support for other tags, pixels etc. Check how to overcome ad blockers etc
