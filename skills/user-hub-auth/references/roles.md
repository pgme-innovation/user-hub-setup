# Facilitator pages

People are `"member"` or `"facilitator"`, set in the hub. Protect on the server first, then hide in the UI.

## Server: the real protection

```ts
import { requireFacilitator, requireHubUser } from "@user-hub/auth/convex";

export const closeVoting = mutation({
  args: {},
  handler: async (ctx) => {
    await requireFacilitator(ctx); // throws for members and for nobody signed in
    // …
  },
});

export const myVotes = query({
  args: {},
  handler: async (ctx) => {
    const me = await requireHubUser(ctx); // throws when signed out
    return await ctx.db
      .query("votes")
      .withIndex("by_person", (q) => q.eq("personId", me.id))
      .take(50);
  },
});
```

TanStack Start server functions and server routes (behind `hub.serverFn` / `hub.request`, see [server-functions.md](server-functions.md)): check `context.hub.role`, and keep `requireFacilitator(ctx)` in any Convex function `context.convex` calls:

```ts
export const resetEvent = createServerFn({ method: "POST" })
  .middleware([hub.serverFn])
  .handler(async ({ context }) => {
    if (context.hub.role !== "facilitator") {
      throw new Error("This is for facilitators.");
    }
    // …
  });
```

Any other server: check `(await verifyHubToken(request, { app })).role`.

## UI: only for a clean screen

```tsx
const me = useHubUser();
if (me?.role !== "facilitator") return <p>This page is for facilitators.</p>;
```

Members can still call the functions behind a hidden page, which is why the server check comes first.
