# FUTURE_FS_03
REBRAND A FAMOUS BRAND'S WEBSITE USING AI

# Frontend Code
Main App Component (client/src/App.tsx):
import { Switch, Route } from "wouter";
import { queryClient } from "./lib/queryClient";
import { QueryClientProvider } from "@tanstack/react-query";
import { Toaster } from "@/components/ui/toaster";
import { TooltipProvider } from "@/components/ui/tooltip";
import { ThemeProvider } from "@/components/theme-provider";
import Home from "@/pages/home";
import Redesign from "@/pages/redesign";
import NotFound from "@/pages/not-found";

function Router() {
  return (
    <Switch>
      <Route path="/" component={Home} />
      <Route path="/redesign" component={Redesign} />
      <Route component={NotFound} />
    </Switch>
  );
}

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider defaultTheme="light">
        <TooltipProvider>
          <Toaster />
          <Router />
        </TooltipProvider>
      </ThemeProvider>
    </QueryClientProvider>
  );
}

export default App;

# Theme Provider (client/src/components/theme-provider.tsx):
import { createContext, useContext, useEffect, useState } from "react";

type Theme = "light" | "dark";

type ThemeProviderProps = {
  children: React.ReactNode;
  defaultTheme?: Theme;
  storageKey?: string;
};

type ThemeProviderState = {
  theme: Theme;
  setTheme: (theme: Theme) => void;
};

const initialState: ThemeProviderState = {
  theme: "light",
  setTheme: () => null,
};

const ThemeProviderContext = createContext<ThemeProviderState>(initialState);

export function ThemeProvider({
  children,
  defaultTheme = "light",
  storageKey = "rebrand-ai-theme",
  ...props
}: ThemeProviderProps) {
  const [theme, setTheme] = useState<Theme>(
    () => (localStorage.getItem(storageKey) as Theme) || defaultTheme
  );

  useEffect(() => {
    const root = window.document.documentElement;
    
    root.classList.remove("light", "dark");
    root.classList.add(theme);
  }, [theme]);

  const value = {
    theme,
    setTheme: (theme: Theme) => {
      localStorage.setItem(storageKey, theme);
      setTheme(theme);
    },
  };

  return (
    <ThemeProviderContext.Provider {...props} value={value}>
      {children}
    </ThemeProviderContext.Provider>
  );
}

export const useTheme = () => {
  const context = useContext(ThemeProviderContext);

  if (context === undefined)
    throw new Error("useTheme must be used within a ThemeProvider");

  return context;
};

# Backend Code
Main Server (server/index.ts):
import express, { type Request, Response, NextFunction } from "express";
import { registerRoutes } from "./routes";
import { setupVite, serveStatic } from "./vite";

const app = express();
app.use(express.json());
app.use(express.urlencoded({ extended: false }));

// Error handling middleware
app.use((err: any, _req: Request, res: Response, _next: NextFunction) => {
  const status = err.status || err.statusCode || 500;
  const message = err.message || "Internal Server Error";

  res.status(status).json({ message });
});

async function main() {
  const server = await registerRoutes(app);

  // Setup Vite or serve static files
  if (app.get("env") === "development") {
    await setupVite(app, server);
  } else {
    serveStatic(app);
  }

  const PORT = 5000;
  server.listen(PORT, "0.0.0.0", () => {
    console.log(`serving on port ${PORT}`);
  });
}

main();

# API Routes (server/routes.ts):
import type { Express } from "express";
import { createServer, type Server } from "http";
import { storage } from "./storage";
import OpenAI from "openai";

// the newest OpenAI model is "gpt-4o" which was released May 13, 2024. do not change this unless explicitly requested by the user
const MODEL = "gpt-4o";

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY || "sk-placeholder-key-for-development"
});

export async function registerRoutes(app: Express): Promise<Server> {
  // API routes for brands
  app.get("/api/brands", async (req, res) => {
    try {
      const category = req.query.category as string | undefined;
      const brands = await storage.getBrands(category);
      res.json(brands);
    } catch (error) {
      console.error("Error fetching brands:", error);
      res.status(500).json({ message: "Failed to fetch brands" });
    }
  });

  app.get("/api/brands/:id", async (req, res) => {
    try {
      const brandId = parseInt(req.params.id);
      const brand = await storage.getBrand(brandId);
      
      if (!brand) {
        return res.status(404).json({ message: "Brand not found" });
      }
      
      res.json(brand);
    } catch (error) {
      console.error("Error fetching brand:", error);
      res.status(500).json({ message: "Failed to fetch brand" });
    }
  });

  // AI API routes
  app.post("/api/ai/generate-color-scheme", async (req, res) => {
    try {
      const { brand, preferences } = req.body;
      
      if (!brand) {
        return res.status(400).json({ 
          success: false, 
          message: "Brand data is required" 
        });
      }

      const prompt = `Generate a modern color scheme for ${brand.name}, a ${brand.industry} brand.
      The original brand color is ${brand.primaryColor || "unknown"}.
      ${preferences?.style ? `The preferred style is: ${preferences.style}.` : ""}
      ${preferences?.mood ? `The preferred mood is: ${preferences.mood}.` : ""}
      Provide a JSON response in this format:
      {
        "primary": "#hex",
        "secondary": "#hex", 
        "accent": "#hex",
        "background": "#hex",
        "text": "#hex",
        "additionalColors": ["#hex", "#hex"]
      }`;

      const response = await openai.chat.completions.create({
        model: MODEL,
        messages: [
          { role: "system", content: "You are a design expert specializing in brand color schemes." },
          { role: "user", content: prompt }
        ],
        response_format: { type: "json_object" }
      });

      const colorScheme = JSON.parse(response.choices[0].message.content || "{}");
      
      res.json({
        success: true,
        data: colorScheme
      });
    } catch (error) {
      console.error("Error generating color scheme:", error);
      res.status(500).json({ 
        success: false, 
        message: "Failed to generate color scheme" 
      });
    }
  });

  // Create HTTP server
  const httpServer = createServer(app);
  return httpServer;
}

# Database Schema (shared/schema.ts):
import { pgTable, text, serial, integer, boolean, jsonb, timestamp } from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";

export const users = pgTable("users", {
  id: serial("id").primaryKey(),
  username: text("username").notNull().unique(),
  password: text("password").notNull(),
  email: text("email"),
  createdAt: timestamp("created_at").defaultNow(),
});

export const brands = pgTable("brands", {
  id: serial("id").primaryKey(),
  name: text("name").notNull(),
  industry: text("industry").notNull(),
  logoUrl: text("logo_url"),
  primaryColor: text("primary_color"),
  website: text("website"),
  category: text("category"),
  backgroundColor: text("background_color"),
  createdAt: timestamp("created_at").defaultNow(),
});

export const projects = pgTable("projects", {
  id: serial("id").primaryKey(),
  userId: integer("user_id").references(() => users.id),
  brandId: integer("brand_id").references(() => brands.id),
  name: text("name").notNull(),
  description: text("description"),
  status: text("status").notNull().default("in_progress"),
  originalDesign: text("original_design"),
  newDesign: text("new_design"),
  colorScheme: jsonb("color_scheme"),
  typography: jsonb("typography"),
  layoutSettings: jsonb("layout_settings"),
  components: jsonb("components"),
  createdAt: timestamp("created_at").defaultNow(),
  updatedAt: timestamp("updated_at").defaultNow(),
});

// Export types
export type InsertUser = z.infer<typeof insertUserSchema>;
export type User = typeof users.$inferSelect;
export type InsertBrand = z.infer<typeof insertBrandSchema>;
export type Brand = typeof brands.$inferSelect;
export type InsertProject = z.infer<typeof insertProjectSchema>;
export type Project = typeof projects.$inferSelect;

// Types for frontend state
export interface ColorScheme {
  primary: string;
  secondary: string;
  accent: string;
  background: string;
  text: string;
  additionalColors?: string[];
}

export interface Typography {
  headingFont: string;
  bodyFont: string;
  fontSize: {
    base: string;
    heading1: string;
    heading2: string;
    heading3: string;
  };
}

# Styling (client/src/index.css):
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --secondary: 326.8 100% 62.7%;
    --secondary-foreground: 210 40% 98%;
    --accent: 24.6 95% 53.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
  }
  
  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 47.4% 11.2%;
  }
  
  body {
    @apply bg-background text-foreground;
    font-family: 'Inter', sans-serif;
  }
  
  .heading {
    font-family: 'Poppins', sans-serif;
  }
  
  .gradient-text {
    background: linear-gradient(90deg, #3B82F6, #EC4899);
    -webkit-background-clip: text;
    background-clip: text;
    color: transparent;
  }
  
  .gradient-bg {
    background: linear-gradient(120deg, #3B82F6, #EC4899);
  }
}
