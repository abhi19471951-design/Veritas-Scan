Data schema:
import { pgTable, text, serial, integer, timestamp, jsonb } from "drizzle-orm/pg-core";
import { createInsertSchema } from "drizzle-zod";
import { z } from "zod";

export const analyses = pgTable("analyses", {
  id: serial("id").primaryKey(),
  content: text("content").notNull(),
  verdict: text("verdict").notNull(), // "likely_real", "likely_fake", "uncertain"
  score: integer("score").notNull(), // 0-100 credibility score
  explanation: text("explanation").notNull(),
  analysisJson: jsonb("analysis_json").notNull(), // Store full breakdown
  createdAt: timestamp("created_at").defaultNow(),
});

export const insertAnalysisSchema = createInsertSchema(analyses).omit({ 
  id: true, 
  createdAt: true 
});

export type Analysis = typeof analyses.$inferSelect;
export type InsertAnalysis = z.infer<typeof insertAnalysisSchema>;

export type CreateAnalysisRequest = {
  content: string;
};

export type AnalysisResponse = Analysis;

Api Logic:
import { z } from 'zod';
import { insertAnalysisSchema, analyses } from './schema';

export const errorSchemas = {
  validation: z.object({
    message: z.string(),
    field: z.string().optional(),
  }),
  notFound: z.object({
    message: z.string(),
  }),
  internal: z.object({
    message: z.string(),
  }),
};

export const api = {
  analyses: {
    create: {
      method: 'POST' as const,
      path: '/api/analyze',
      input: z.object({
        content: z.string().min(10, "Content must be at least 10 characters long").max(5000, "Content too long"),
      }),
      responses: {
        200: z.custom<typeof analyses.$inferSelect>(),
        400: errorSchemas.validation,
        500: errorSchemas.internal,
      },
    },
    list: {
      method: 'GET' as const,
      path: '/api/analyses',
      responses: {
        200: z.array(z.custom<typeof analyses.$inferSelect>()),
      },
    },
  },
};

export function buildUrl(path: string, params?: Record<string, string | number>): string {
  let url = path;
  if (params) {
    Object.entries(params).forEach(([key, value]) => {
      if (url.includes(`:${key}`)) {
        url = url.replace(`:${key}`, String(value));
      }
    });
  }
  return url;
}

export type CreateAnalysisInput = z.infer<typeof api.analyses.create.input>;
export type AnalysisResponse = z.infer<typeof api.analyses.create.responses[200]>;

Server Route:
import type { Express } from "express";
import { type Server } from "http";
import { storage } from "./storage";
import { api } from "@shared/routes";
import { z } from "zod";
import OpenAI from "openai";
import { db } from "./db";
import { analyses } from "@shared/schema";
import { count } from "drizzle-orm";

const openai = new OpenAI({
  apiKey: process.env.AI_INTEGRATIONS_OPENAI_API_KEY,
  baseURL: process.env.AI_INTEGRATIONS_OPENAI_BASE_URL,
});

export async function registerRoutes(
  httpServer: Server,
  app: Express
): Promise<Server> {
  app.post(api.analyses.create.path, async (req, res) => {
    try {
      const input = api.analyses.create.input.parse(req.body);

      const completion = await openai.chat.completions.create({
        model: "gpt-4o",
        messages: [
          {
            role: "system",
            content: `You are an expert fact-checker and credibility analyst. 
            Today is December 22, 2025. You have access to real-time information.
            
            Analyze the provided text for misinformation, emotional manipulation, and sensationalism.
            
            Return a JSON object:
            {
              "verdict": "likely_real" | "likely_fake" | "uncertain",
              "score": number (0-100),
              "explanation": "Concise summary.",
              "analysis_json": { ... }
            }`
          },
          { role: "user", content: input.content }
        ],
        response_format: { type: "json_object" },
      });

      const result = JSON.parse(completion.choices[0].message.content || "{}");

      const analysis = await storage.createAnalysis({
        content: input.content,
        verdict: result.verdict,
        score: result.score,
        explanation: result.explanation,
        analysisJson: result.analysis_json || {},
      });

      res.json(analysis);
    } catch (err) {
      res.status(500).json({ message: "Failed to analyze content" });
    }
  });

  return httpServer;
}
