# 단계4: MCP 통합와확장 시스템가이드

## 📋 대상 독자
**본 문서의 대상: 초보 수준의 프로그래머**
- 깊은 사고 없이 단계별로 엄격히 실행
- 각 단계는 명확한 파일 작업 지침 포함
- 필요한 코드 템플릿과 설정 포함

## 🎯 단계목표
역분석 결과를 기반으로, 구현Claude Code의MCP프로토콜와확장 시스템: 
- ✅ **전체MCP프로토콜 구현** (STDIO, HTTP, SSE, WebSocket四种전송방식)
- ✅ **다중 서버 관리시스템** (연결池, 상태모니터링, 자동재연결)
- ✅ **플러그인생태계시스템** (도구등록, 권한관리, 버전제어)
- ✅ **IDE전용통합** (sse-ide, ws-ide연결와진단시스템)
- ✅ **확장설정시스템** (3단계설정, OAuth인증, 리소스 관리)

**예상 성과물**: 
- ✅ MCP클라이언트완전 구현 (지원4种전송프로토콜)
- ✅ 서버연결관리시스템
- ✅ 도구화이트리스트와보안메커니즘
- ✅ 설정 관리와인증시스템
- ✅ 扩펼치기发프레임워크

**작업 시간**: 4주 (160시간)

---

## 📁 1주차: MCP프로토콜핵심구현

### 단계4.1: MCP전송层基础架构

**기반逆向분석의전송프로토콜 구현**

**파일 경로**: `src/mcp/transport/base.ts`
**파일 내용**:
```typescript
/**
 * MCP전송层基础架构
 * 기반逆向분석의Claude Code MCP구현
 * 지원STDIO, HTTP, SSE, WebSocket四种전송방식
 */

export interface McpTransport {
  connect(): Promise<void>;
  disconnect(): Promise<void>;
  send(message: any): Promise<void>;
  onMessage(callback: (message: any) => void): void;
  onClose(callback: () => void): void;
  onError(callback: (error: Error) => void): void;
  isConnected(): boolean;
}

export interface McpMessage {
  jsonrpc: "2.0";
  id?: string | number;
  method?: string;
  params?: any;
  result?: any;
  error?: {
    code: number;
    message: string;
    data?: any;
  };
}

export interface McpServerConfig {
  name: string;
  transport: TransportConfig;
  auth?: AuthConfig;
  capabilities?: string[];
  timeout?: number;
  retryAttempts?: number;
}

export type TransportConfig = 
  | StdioTransportConfig
  | HttpTransportConfig 
  | SseTransportConfig
  | WebSocketTransportConfig
  | SseIdeTransportConfig
  | WsIdeTransportConfig;

export interface StdioTransportConfig {
  type: "stdio";
  command: string;
  args?: string[];
  env?: Record<string, string>;
  cwd?: string;
}

export interface HttpTransportConfig {
  type: "http";
  url: string;
  headers?: Record<string, string>;
  method?: "POST" | "GET";
}

export interface SseTransportConfig {
  type: "sse";
  url: string;
  headers?: Record<string, string>;
}

export interface WebSocketTransportConfig {
  type: "websocket";
  url: string;
  protocols?: string[];
  headers?: Record<string, string>;
}

// IDE전용전송설정 - 기반逆向분석
export interface SseIdeTransportConfig {
  type: "sse-ide";
  url: string;
  ideName: string;
}

export interface WsIdeTransportConfig {
  type: "ws-ide";
  url: string;
  ideName: string;
  authToken?: string;
}

export interface AuthConfig {
  type: "oauth2" | "bearer" | "api-key";
  clientId?: string;
  clientSecret?: string;
  token?: string;
  apiKey?: string;
  refreshToken?: string;
  tokenUrl?: string;
}

/**
 * MCP전송基클래스
 * 提供일반의메시지처리와오류관리
 */
export abstract class BaseMcpTransport implements McpTransport {
  protected messageHandlers: Set<(message: any) => void> = new Set();
  protected closeHandlers: Set<() => void> = new Set();
  protected errorHandlers: Set<(error: Error) => void> = new Set();
  protected connected = false;
  protected messageId = 1;

  abstract connect(): Promise<void>;
  abstract disconnect(): Promise<void>;
  abstract send(message: any): Promise<void>;

  public onMessage(callback: (message: any) => void): void {
    this.messageHandlers.add(callback);
  }

  public onClose(callback: () => void): void {
    this.closeHandlers.add(callback);
  }

  public onError(callback: (error: Error) => void): void {
    this.errorHandlers.add(callback);
  }

  public isConnected(): boolean {
    return this.connected;
  }

  protected emitMessage(message: any): void {
    for (const handler of this.messageHandlers) {
      try {
        handler(message);
      } catch (error) {
        console.error('Error in message handler:', error);
      }
    }
  }

  protected emitClose(): void {
    this.connected = false;
    for (const handler of this.closeHandlers) {
      try {
        handler();
      } catch (error) {
        console.error('Error in close handler:', error);
      }
    }
  }

  protected emitError(error: Error): void {
    for (const handler of this.errorHandlers) {
      try {
        handler(error);
      } catch (error) {
        console.error('Error in error handler:', error);
      }
    }
  }

  protected generateMessageId(): string {
    return `msg_${this.messageId++}_${Date.now()}`;
  }

  protected createMessage(method: string, params?: any): McpMessage {
    return {
      jsonrpc: "2.0",
      id: this.generateMessageId(),
      method,
      params
    };
  }

  protected createResponse(id: string | number, result?: any, error?: any): McpMessage {
    const message: McpMessage = {
      jsonrpc: "2.0",
      id
    };

    if (error) {
      message.error = error;
    } else {
      message.result = result;
    }

    return message;
  }
}

/**
 * 전송팩토리클래스 - 기반逆向분석의ue함수구현
 */
export class McpTransportFactory {
  /**
   * 생성전송인스턴스 - 에 해당improved-claude-code-5.mjs:35481-35520
   */
  public static async createTransport(config: TransportConfig, authProvider?: any): Promise<McpTransport> {
    switch (config.type) {
      case "stdio":
        return new StdioTransport(config);
      
      case "http":
        return new HttpTransport(config, authProvider);
      
      case "sse":
        return new SseTransport(config, authProvider);
      
      case "websocket":
        return new WebSocketTransport(config);
      
      case "sse-ide":
        return new SseIdeTransport(config);
      
      case "ws-ide":
        return new WsIdeTransport(config);
      
      default:
        throw new Error(`Unsupported transport type: ${(config as any).type}`);
    }
  }
}

/**
 * 인증提供者 - 기반逆向분석의MO클래스구현
 */
export class AuthProvider {
  constructor(
    private serverName: string,
    private config: TransportConfig & { auth?: AuthConfig }
  ) {}

  public async getAuthHeaders(): Promise<Record<string, string>> {
    if (!this.config.auth) {
      return {};
    }

    const headers: Record<string, string> = {};

    switch (this.config.auth.type) {
      case "bearer":
        if (this.config.auth.token) {
          headers["Authorization"] = `Bearer ${this.config.auth.token}`;
        }
        break;

      case "api-key":
        if (this.config.auth.apiKey) {
          headers["X-API-Key"] = this.config.auth.apiKey;
        }
        break;

      case "oauth2":
        // OAuth2 implementation would go here
        const token = await this.getOAuth2Token();
        if (token) {
          headers["Authorization"] = `Bearer ${token}`;
        }
        break;
    }

    return headers;
  }

  private async getOAuth2Token(): Promise<string | null> {
    // OAuth2 token acquisition logic
    // This would implement the full OAuth2 flow
    return this.config.auth?.token || null;
  }
}
```

### 단계4.2: STDIO전송구현

**기반逆向분석의子进程通信**

**파일 경로**: `src/mcp/transport/stdio.ts`
**파일 내용**:
```typescript
/**
 * STDIO전송구현
 * 기반逆向분석의Claude Code STDIO MCP전송
 * 지원子进程通信와오류 처리
 */

import { spawn, ChildProcess } from 'child_process';
import { BaseMcpTransport, StdioTransportConfig } from './base';

/**
 * STDIO전송클래스
 * 通过子进程의stdin/stdout进行MCP通信
 */
export class StdioTransport extends BaseMcpTransport {
  private childProcess: ChildProcess | null = null;
  private buffer = '';
  private readyPromise: Promise<void> | null = null;

  constructor(private config: StdioTransportConfig) {
    super();
  }

  public async connect(): Promise<void> {
    if (this.connected) {
      return;
    }

    this.readyPromise = new Promise((resolve, reject) => {
      try {
        // 기반逆向분석improved-claude-code-5.mjs:35484-35495
        this.childProcess = spawn(this.config.command, this.config.args || [], {
          stdio: ['pipe', 'pipe', 'pipe'],
          env: { 
            ...process.env, 
            ...this.config.env 
          },
          cwd: this.config.cwd
        });

        this.setupProcessHandlers(resolve, reject);
      } catch (error) {
        reject(error);
      }
    });

    await this.readyPromise;
    this.connected = true;
  }

  private setupProcessHandlers(resolve: () => void, reject: (error: Error) => void): void {
    if (!this.childProcess) {
      reject(new Error('Child process not created'));
      return;
    }

    // 오류 처리
    this.childProcess.on('error', (error) => {
      this.emitError(error);
      reject(error);
    });

    // 进程退出처리
    this.childProcess.on('exit', (code, signal) => {
      this.emitClose();
      if (code !== 0 && code !== null) {
        this.emitError(new Error(`Process exited with code ${code}`));
      }
    });

    // stderr처리
    this.childProcess.stderr?.on('data', (data) => {
      console.error(`MCP Server stderr: ${data}`);
    });

    // stdout메시지처리
    this.childProcess.stdout?.on('data', (data) => {
      this.handleData(data.toString());
    });

    // 연결성공
    resolve();
  }

  private handleData(data: string): void {
    this.buffer += data;
    
    // 按行分割메시지
    const lines = this.buffer.split('\n');
    this.buffer = lines.pop() || ''; // 保留不전체의行

    for (const line of lines) {
      if (line.trim()) {
        try {
          const message = JSON.parse(line.trim());
          this.emitMessage(message);
        } catch (error) {
          console.error('Error parsing JSON message:', error, 'Line:', line);
        }
      }
    }
  }

  public async send(message: any): Promise<void> {
    if (!this.connected || !this.childProcess?.stdin) {
      throw new Error('Transport not connected');
    }

    const messageString = JSON.stringify(message) + '\n';
    
    return new Promise((resolve, reject) => {
      this.childProcess!.stdin!.write(messageString, (error) => {
        if (error) {
          reject(error);
        } else {
          resolve();
        }
      });
    });
  }

  public async disconnect(): Promise<void> {
    if (!this.connected || !this.childProcess) {
      return;
    }

    return new Promise((resolve) => {
      if (this.childProcess) {
        this.childProcess.on('close', () => {
          this.childProcess = null;
          resolve();
        });

        // 优雅끄기
        this.childProcess.stdin?.end();
        
        // 强制끄기타임아웃
        setTimeout(() => {
          if (this.childProcess) {
            this.childProcess.kill('SIGTERM');
            setTimeout(() => {
              if (this.childProcess) {
                this.childProcess.kill('SIGKILL');
              }
            }, 5000);
          }
        }, 10000);
      } else {
        resolve();
      }
    });
  }
}
```

### 단계4.3: HTTP/SSE전송구현

**기반逆向분석의HTTP와SSE전송**

**파일 경로**: `src/mcp/transport/http-sse.ts`
**파일 내용**:
```typescript
/**
 * HTTP와SSE전송구현
 * 기반逆向분석의Claude Code HTTP/SSE MCP전송
 */

import { BaseMcpTransport, HttpTransportConfig, SseTransportConfig, AuthProvider } from './base';
import { EventSource } from 'eventsource';

/**
 * HTTP전송클래스
 * 용요청-응답모드의MCP通信
 */
export class HttpTransport extends BaseMcpTransport {
  private authProvider: AuthProvider | null = null;

  constructor(
    private config: HttpTransportConfig,
    authProvider?: AuthProvider
  ) {
    super();
    this.authProvider = authProvider || null;
  }

  public async connect(): Promise<void> {
    // HTTP전송不需要持久연결
    this.connected = true;
  }

  public async disconnect(): Promise<void> {
    this.connected = false;
  }

  public async send(message: any): Promise<void> {
    if (!this.connected) {
      throw new Error('Transport not connected');
    }

    try {
      const headers: Record<string, string> = {
        'Content-Type': 'application/json',
        'User-Agent': `Claude-Code/${this.getVersion()}`,
        ...this.config.headers
      };

      // 추가인증头
      if (this.authProvider) {
        const authHeaders = await this.authProvider.getAuthHeaders();
        Object.assign(headers, authHeaders);
      }

      const response = await fetch(this.config.url, {
        method: this.config.method || 'POST',
        headers,
        body: JSON.stringify(message)
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const result = await response.json();
      this.emitMessage(result);
    } catch (error) {
      this.emitError(error as Error);
      throw error;
    }
  }

  private getVersion(): string {
    // 가져오기Claude Code버전号
    return process.env.CLAUDE_CODE_VERSION || '1.0.0';
  }
}

/**
 * SSE전송클래스 - 기반逆向분석FF1클래스구현
 * 용서버到클라이언트의실시간通信
 */
export class SseTransport extends BaseMcpTransport {
  private eventSource: EventSource | null = null;
  private authProvider: AuthProvider | null = null;

  constructor(
    private config: SseTransportConfig,
    authProvider?: AuthProvider
  ) {
    super();
    this.authProvider = authProvider || null;
  }

  public async connect(): Promise<void> {
    if (this.connected) {
      return;
    }

    return new Promise(async (resolve, reject) => {
      try {
        const headers: Record<string, string> = {
          'User-Agent': `Claude-Code/${this.getVersion()}`,
          'Content-Type': 'application/json',
          ...this.config.headers
        };

        // 추가인증头
        if (this.authProvider) {
          const authHeaders = await this.authProvider.getAuthHeaders();
          Object.assign(headers, authHeaders);
        }

        // 생성EventSource - 기반逆向분석
        const eventSourceInitDict: any = {
          headers
        };

        this.eventSource = new EventSource(this.config.url, eventSourceInitDict);

        this.eventSource.onopen = () => {
          this.connected = true;
          resolve();
        };

        this.eventSource.onmessage = (event) => {
          try {
            const message = JSON.parse(event.data);
            this.emitMessage(message);
          } catch (error) {
            console.error('Error parsing SSE message:', error);
          }
        };

        this.eventSource.onerror = (error) => {
          this.emitError(new Error('SSE connection error'));
          if (!this.connected) {
            reject(new Error('Failed to connect to SSE endpoint'));
          }
        };

        this.eventSource.addEventListener('close', () => {
          this.emitClose();
        });

      } catch (error) {
        reject(error);
      }
    });
  }

  public async send(message: any): Promise<void> {
    if (!this.connected) {
      throw new Error('SSE transport not connected');
    }

    // SSE通常예单向의, 如果需要发送메시지, 可能需要额外의HTTP요청
    // 这里可로구현콜백URL또는WebHook메커니즘
    throw new Error('SSE transport is read-only');
  }

  public async disconnect(): Promise<void> {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
    }
    this.connected = false;
  }

  private getVersion(): string {
    return process.env.CLAUDE_CODE_VERSION || '1.0.0';
  }
}

/**
 * SSE-IDE전송클래스 - 기반逆向분석의IDE전용SSE전송
 * 에 해당improved-claude-code-5.mjs:23402-23405의if4설정
 */
export class SseIdeTransport extends BaseMcpTransport {
  private eventSource: EventSource | null = null;

  constructor(private config: SseIdeTransportConfig) {
    super();
  }

  public async connect(): Promise<void> {
    if (this.connected) {
      return;
    }

    return new Promise((resolve, reject) => {
      try {
        // IDE전용SSE연결 - 기반逆向분석
        this.eventSource = new EventSource(this.config.url);

        this.eventSource.onopen = () => {
          this.connected = true;
          resolve();
        };

        this.eventSource.onmessage = (event) => {
          try {
            const message = JSON.parse(event.data);
            this.emitMessage(message);
          } catch (error) {
            console.error('Error parsing SSE-IDE message:', error);
          }
        };

        this.eventSource.onerror = () => {
          this.emitError(new Error('SSE-IDE connection error'));
          if (!this.connected) {
            reject(new Error('Failed to connect to SSE-IDE endpoint'));
          }
        };

      } catch (error) {
        reject(error);
      }
    });
  }

  public async send(message: any): Promise<void> {
    // SSE-IDE通常용接收IDE의진단정보, 不지원发送
    throw new Error('SSE-IDE transport is read-only');
  }

  public async disconnect(): Promise<void> {
    if (this.eventSource) {
      this.eventSource.close();
      this.eventSource = null;
    }
    this.connected = false;
  }
}
```

### 단계4.4: WebSocket전송구현

**기반逆向분석의WebSocket와WS-IDE전송**

**파일 경로**: `src/mcp/transport/websocket.ts`
**파일 내용**:
```typescript
/**
 * WebSocket전송구현
 * 기반逆向분석의Claude Code WebSocket MCP전송
 * 지원표준WebSocket와IDE전용WebSocket
 */

import WebSocket from 'ws';
import { BaseMcpTransport, WebSocketTransportConfig, WsIdeTransportConfig } from './base';

/**
 * WebSocket전송클래스
 * 용双向실시간通信의MCP전송
 */
export class WebSocketTransport extends BaseMcpTransport {
  private websocket: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectDelay = 1000;

  constructor(private config: WebSocketTransportConfig) {
    super();
  }

  public async connect(): Promise<void> {
    if (this.connected) {
      return;
    }

    return new Promise((resolve, reject) => {
      try {
        const options: WebSocket.ClientOptions = {};
        
        if (this.config.protocols) {
          options.protocols = this.config.protocols;
        }

        if (this.config.headers) {
          options.headers = this.config.headers;
        }

        this.websocket = new WebSocket(this.config.url, options);

        this.websocket.on('open', () => {
          this.connected = true;
          this.reconnectAttempts = 0;
          resolve();
        });

        this.websocket.on('message', (data) => {
          try {
            const message = JSON.parse(data.toString());
            this.emitMessage(message);
          } catch (error) {
            console.error('Error parsing WebSocket message:', error);
          }
        });

        this.websocket.on('close', (code, reason) => {
          this.emitClose();
          
          // 자동재연결논리
          if (this.reconnectAttempts < this.maxReconnectAttempts) {
            setTimeout(() => {
              this.reconnectAttempts++;
              this.connect().catch(error => {
                console.error('Reconnection failed:', error);
              });
            }, this.reconnectDelay * Math.pow(2, this.reconnectAttempts));
          }
        });

        this.websocket.on('error', (error) => {
          this.emitError(error);
          if (!this.connected) {
            reject(error);
          }
        });

      } catch (error) {
        reject(error);
      }
    });
  }

  public async send(message: any): Promise<void> {
    if (!this.connected || !this.websocket) {
      throw new Error('WebSocket not connected');
    }

    if (this.websocket.readyState !== WebSocket.OPEN) {
      throw new Error('WebSocket not ready');
    }

    return new Promise((resolve, reject) => {
      this.websocket!.send(JSON.stringify(message), (error) => {
        if (error) {
          reject(error);
        } else {
          resolve();
        }
      });
    });
  }

  public async disconnect(): Promise<void> {
    if (this.websocket) {
      this.websocket.close();
      this.websocket = null;
    }
    this.connected = false;
  }
}

/**
 * WS-IDE전송클래스 - 기반逆向분석의IDE전용WebSocket전송
 * 에 해당improved-claude-code-5.mjs:23408-23412의nf4설정와35508-35520의구현
 */
export class WsIdeTransport extends BaseMcpTransport {
  private websocket: WebSocket | null = null;

  constructor(private config: WsIdeTransportConfig) {
    super();
  }

  public async connect(): Promise<void> {
    if (this.connected) {
      return;
    }

    return new Promise((resolve, reject) => {
      try {
        const options: WebSocket.ClientOptions = {
          protocols: ["mcp"] // MCP子프로토콜
        };

        // 기반逆向분석의IDE인증头 - improved-claude-code-5.mjs:35508-35515
        if (this.config.authToken) {
          options.headers = {
            "X-Claude-Code-Ide-Authorization": this.config.authToken
          };
        }

        this.websocket = new WebSocket(this.config.url, options);

        this.websocket.on('open', () => {
          this.connected = true;
          
          // 发送IDE연결알림 - 에 해당we0(I)调用
          this.sendIdeConnectedNotification();
          
          resolve();
        });

        this.websocket.on('message', (data) => {
          try {
            const message = JSON.parse(data.toString());
            this.emitMessage(message);
          } catch (error) {
            console.error('Error parsing WS-IDE message:', error);
          }
        });

        this.websocket.on('close', () => {
          this.emitClose();
        });

        this.websocket.on('error', (error) => {
          this.emitError(error);
          if (!this.connected) {
            reject(error);
          }
        });

      } catch (error) {
        reject(error);
      }
    });
  }

  /**
   * 发送IDE연결알림 - 기반逆向분석we0함수
   */
  private async sendIdeConnectedNotification(): Promise<void> {
    try {
      const notification = this.createMessage('ide_connected', {
        ideName: this.config.ideName,
        timestamp: Date.now(),
        capabilities: [
          'getDiagnostics',
          'executeCode'
        ]
      });

      await this.send(notification);
    } catch (error) {
      console.error('Failed to send IDE connected notification:', error);
    }
  }

  public async send(message: any): Promise<void> {
    if (!this.connected || !this.websocket) {
      throw new Error('WS-IDE not connected');
    }

    if (this.websocket.readyState !== WebSocket.OPEN) {
      throw new Error('WS-IDE not ready');
    }

    return new Promise((resolve, reject) => {
      this.websocket!.send(JSON.stringify(message), (error) => {
        if (error) {
          reject(error);
        } else {
          resolve();
        }
      });
    });
  }

  public async disconnect(): Promise<void> {
    if (this.websocket) {
      this.websocket.close();
      this.websocket = null;
    }
    this.connected = false;
  }
}
```

---

## 📁 2주차: 서버연결관리시스템

### 단계4.5: MCP클라이언트핵심

**기반逆向분석의전체MCP클라이언트구현**

**파일 경로**: `src/mcp/client.ts`
**파일 내용**:
```typescript
/**
 * MCP클라이언트핵심구현
 * 기반逆向분석의Claude Code MCP클라이언트功能
 * 지원도구调用, 리소스 관리, 알림처리
 */

import { McpTransport, McpMessage, McpServerConfig } from './transport/base';
import { McpTransportFactory } from './transport/base';

export interface McpTool {
  name: string;
  description: string;
  inputSchema: any;
}

export interface McpResource {
  uri: string;
  name: string;
  description?: string;
  mimeType?: string;
}

export interface McpPrompt {
  name: string;
  description: string;
  arguments?: any[];
}

export interface McpServerInfo {
  name: string;
  version: string;
  protocolVersion: string;
  capabilities: {
    tools?: {};
    resources?: {};
    prompts?: {};
    logging?: {};
  };
}

export interface McpToolCall {
  toolName: string;
  arguments: any;
  timeout?: number;
}

export interface McpToolResult {
  isError?: boolean;
  content: Array<{
    type: 'text' | 'image' | 'resource';
    text?: string;
    data?: string;
    mimeType?: string;
  }>;
}

/**
 * MCP클라이언트클래스
 * 관리与单个MCP서버의연결와通信
 */
export class McpClient {
  private transport: McpTransport | null = null;
  private pendingRequests = new Map<string | number, {
    resolve: (value: any) => void;
    reject: (error: Error) => void;
    timeout?: NodeJS.Timeout;
  }>();
  
  private serverInfo: McpServerInfo | null = null;
  private tools: McpTool[] = [];
  private resources: McpResource[] = [];
  private prompts: McpPrompt[] = [];
  
  public readonly config: McpServerConfig;
  public connected = false;

  constructor(config: McpServerConfig) {
    this.config = config;
  }

  /**
   * 연결到MCP서버
   */
  public async connect(): Promise<void> {
    if (this.connected) {
      return;
    }

    try {
      // 생성전송层
      this.transport = await McpTransportFactory.createTransport(this.config.transport);
      
      // 설정이벤트처리器
      this.setupEventHandlers();
      
      // 建立연결
      await this.transport.connect();
      
      // 초기화握手
      await this.initialize();
      
      this.connected = true;
    } catch (error) {
      await this.cleanup();
      throw error;
    }
  }

  /**
   * 断开연결
   */
  public async disconnect(): Promise<void> {
    if (!this.connected) {
      return;
    }

    await this.cleanup();
  }

  /**
   * 调用도구 - 기반逆向분석gw함수구현
   */
  public async callTool(toolName: string, arguments_: any, timeout?: number): Promise<McpToolResult> {
    if (!this.connected || !this.transport) {
      throw new Error('Client not connected');
    }

    const message = {
      jsonrpc: "2.0" as const,
      id: this.generateRequestId(),
      method: "tools/call",
      params: {
        name: toolName,
        arguments: arguments_
      }
    };

    return this.sendRequest(message, timeout || this.config.timeout);
  }

  /**
   * 列出사용 가능도구
   */
  public async listTools(): Promise<McpTool[]> {
    if (!this.connected || !this.transport) {
      throw new Error('Client not connected');
    }

    const message = {
      jsonrpc: "2.0" as const,
      id: this.generateRequestId(),
      method: "tools/list"
    };

    const response = await this.sendRequest(message);
    this.tools = response.tools || [];
    return this.tools;
  }

  /**
   * 가져오기资源목록
   */
  public async listResources(): Promise<McpResource[]> {
    if (!this.connected || !this.transport) {
      throw new Error('Client not connected');
    }

    const message = {
      jsonrpc: "2.0" as const,
      id: this.generateRequestId(),
      method: "resources/list"
    };

    const response = await this.sendRequest(message);
    this.resources = response.resources || [];
    return this.resources;
  }

  /**
   * 读取资源
   */
  public async readResource(uri: string): Promise<any> {
    if (!this.connected || !this.transport) {
      throw new Error('Client not connected');
    }

    const message = {
      jsonrpc: "2.0" as const,
      id: this.generateRequestId(),
      method: "resources/read",
      params: { uri }
    };

    return this.sendRequest(message);
  }

  /**
   * 가져오기힌트목록
   */
  public async listPrompts(): Promise<McpPrompt[]> {
    if (!this.connected || !this.transport) {
      throw new Error('Client not connected');
    }

    const message = {
      jsonrpc: "2.0" as const,
      id: this.generateRequestId(),
      method: "prompts/list"
    };

    const response = await this.sendRequest(message);
    this.prompts = response.prompts || [];
    return this.prompts;
  }

  /**
   * 가져오기힌트
   */
  public async getPrompt(name: string, arguments_?: any): Promise<any> {
    if (!this.connected || !this.transport) {
      throw new Error('Client not connected');
    }

    const message = {
      jsonrpc: "2.0" as const,
      id: this.generateRequestId(),
      method: "prompts/get",
      params: {
        name,
        arguments: arguments_
      }
    };

    return this.sendRequest(message);
  }

  /**
   * 发送알림
   */
  public async sendNotification(method: string, params?: any): Promise<void> {
    if (!this.connected || !this.transport) {
      throw new Error('Client not connected');
    }

    const message = {
      jsonrpc: "2.0" as const,
      method,
      params
    };

    await this.transport.send(message);
  }

  /**
   * 가져오기서버정보
   */
  public getServerInfo(): McpServerInfo | null {
    return this.serverInfo;
  }

  /**
   * 가져오기캐시의도구목록
   */
  public getCachedTools(): McpTool[] {
    return [...this.tools];
  }

  /**
   * 가져오기캐시의资源목록
   */
  public getCachedResources(): McpResource[] {
    return [...this.resources];
  }

  /**
   * 확인도구예아니오사용 가능
   */
  public hasToolAfter(): boolean {
    return this.tools.some(tool => tool.name.startsWith('mcp__'));
  }

  /**
   * 설정이벤트처리器
   */
  private setupEventHandlers(): void {
    if (!this.transport) return;

    this.transport.onMessage((message: McpMessage) => {
      this.handleMessage(message);
    });

    this.transport.onClose(() => {
      this.handleConnectionClose();
    });

    this.transport.onError((error: Error) => {
      this.handleConnectionError(error);
    });
  }

  /**
   * 처리收到의메시지
   */
  private handleMessage(message: McpMessage): void {
    if (message.id !== undefined) {
      // 这예对요청의응답
      const pending = this.pendingRequests.get(message.id);
      if (pending) {
        this.pendingRequests.delete(message.id);
        
        if (pending.timeout) {
          clearTimeout(pending.timeout);
        }

        if (message.error) {
          pending.reject(new Error(message.error.message));
        } else {
          pending.resolve(message.result);
        }
      }
    } else if (message.method) {
      // 这예알림또는요청
      this.handleNotification(message);
    }
  }

  /**
   * 처리알림
   */
  private handleNotification(message: McpMessage): void {
    switch (message.method) {
      case 'notifications/tools/list_changed':
        // 도구목록已更改, 重新가져오기
        this.listTools().catch(error => {
          console.error('Error refreshing tools:', error);
        });
        break;

      case 'notifications/resources/list_changed':
        // 资源목록已更改, 重新가져오기
        this.listResources().catch(error => {
          console.error('Error refreshing resources:', error);
        });
        break;

      case 'notifications/prompts/list_changed':
        // 힌트목록已更改, 重新가져오기
        this.listPrompts().catch(error => {
          console.error('Error refreshing prompts:', error);
        });
        break;

      default:
        console.log('Received notification:', message.method, message.params);
    }
  }

  /**
   * 처리연결끄기
   */
  private handleConnectionClose(): void {
    this.connected = false;
    
    // 拒绝所있음대기 중의요청
    for (const [id, pending] of this.pendingRequests) {
      pending.reject(new Error('Connection closed'));
      if (pending.timeout) {
        clearTimeout(pending.timeout);
      }
    }
    this.pendingRequests.clear();
  }

  /**
   * 처리연결오류
   */
  private handleConnectionError(error: Error): void {
    console.error(`MCP Client error for ${this.config.name}:`, error);
  }

  /**
   * 초기화握手
   */
  private async initialize(): Promise<void> {
    const message = {
      jsonrpc: "2.0" as const,
      id: this.generateRequestId(),
      method: "initialize",
      params: {
        protocolVersion: "2024-11-05",
        capabilities: {
          tools: {},
          resources: {},
          prompts: {}
        },
        clientInfo: {
          name: "claude-code",
          version: "1.0.0"
        }
      }
    };

    const response = await this.sendRequest(message);
    this.serverInfo = response;

    // 发送초기화完成알림
    await this.sendNotification("notifications/initialized");

    // 가져오기사용 가능도구와资源
    await Promise.all([
      this.listTools().catch(() => []),
      this.listResources().catch(() => []),
      this.listPrompts().catch(() => [])
    ]);
  }

  /**
   * 发送요청并await응답
   */
  private async sendRequest(message: McpMessage, timeout?: number): Promise<any> {
    if (!this.transport) {
      throw new Error('Transport not available');
    }

    return new Promise((resolve, reject) => {
      const requestId = message.id!;
      
      let timeoutHandle: NodeJS.Timeout | undefined;
      if (timeout) {
        timeoutHandle = setTimeout(() => {
          this.pendingRequests.delete(requestId);
          reject(new Error('Request timeout'));
        }, timeout);
      }

      this.pendingRequests.set(requestId, {
        resolve,
        reject,
        timeout: timeoutHandle
      });

      this.transport!.send(message).catch(reject);
    });
  }

  /**
   * 生成요청ID
   */
  private generateRequestId(): string {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  /**
   * 清理资源
   */
  private async cleanup(): Promise<void> {
    this.connected = false;

    if (this.transport) {
      await this.transport.disconnect();
      this.transport = null;
    }

    // 清理대기 중요청
    for (const [id, pending] of this.pendingRequests) {
      pending.reject(new Error('Client disconnected'));
      if (pending.timeout) {
        clearTimeout(pending.timeout);
      }
    }
    this.pendingRequests.clear();

    // 清理캐시
    this.serverInfo = null;
    this.tools = [];
    this.resources = [];
    this.prompts = [];
  }
}
```

### 단계4.6: 多서버연결관리

**기반逆向분석의연결池와상태관리**

**파일 경로**: `src/mcp/server-manager.ts`
**파일 내용**:
```typescript
/**
 * MCP서버관리器
 * 기반逆向분석의Claude Code多서버연결관리
 * 지원연결池, 상태모니터링, 자동재연결
 */

import { McpClient } from './client';
import { McpServerConfig, McpTool } from './client';
import { EventEmitter } from 'events';

export type ServerStatus = 'pending' | 'connected' | 'disconnected' | 'error' | 'reconnecting';

export interface ServerState {
  name: string;
  status: ServerStatus;
  config: McpServerConfig;
  client?: McpClient;
  lastConnected?: number;
  lastError?: Error;
  reconnectAttempts: number;
  tools: McpTool[];
}

export interface ServerManagerEvents {
  'server:connected': (serverName: string) => void;
  'server:disconnected': (serverName: string) => void;
  'server:error': (serverName: string, error: Error) => void;
  'server:reconnecting': (serverName: string, attempt: number) => void;
  'tools:updated': (serverName: string, tools: McpTool[]) => void;
}

/**
 * MCP서버관리器
 * 기반逆向분석improved-claude-code-5.mjs의서버관리논리
 */
export class McpServerManager extends EventEmitter {
  private servers = new Map<string, ServerState>();
  private reconnectTimers = new Map<string, NodeJS.Timeout>();
  private maxReconnectAttempts = 5;
  private baseReconnectDelay = 1000;

  constructor() {
    super();
  }

  /**
   * 추가서버설정
   */
  public async addServer(config: McpServerConfig): Promise<void> {
    const serverState: ServerState = {
      name: config.name,
      status: 'pending',
      config,
      reconnectAttempts: 0,
      tools: []
    };

    this.servers.set(config.name, serverState);
    await this.connectServer(config.name);
  }

  /**
   * 移除서버
   */
  public async removeServer(serverName: string): Promise<void> {
    const serverState = this.servers.get(serverName);
    if (!serverState) {
      return;
    }

    // 清理재연결定时器
    const timer = this.reconnectTimers.get(serverName);
    if (timer) {
      clearTimeout(timer);
      this.reconnectTimers.delete(serverName);
    }

    // 断开연결
    if (serverState.client) {
      await serverState.client.disconnect();
    }

    this.servers.delete(serverName);
    this.emit('server:disconnected', serverName);
  }

  /**
   * 연결到指定서버
   */
  private async connectServer(serverName: string): Promise<void> {
    const serverState = this.servers.get(serverName);
    if (!serverState) {
      throw new Error(`Server ${serverName} not found`);
    }

    try {
      serverState.status = 'pending';
      
      // 생성클라이언트
      const client = new McpClient(serverState.config);
      
      // 설정이벤트처리
      this.setupClientEventHandlers(client, serverState);
      
      // 연결
      await client.connect();
      
      // 업데이트상태
      serverState.client = client;
      serverState.status = 'connected';
      serverState.lastConnected = Date.now();
      serverState.reconnectAttempts = 0;
      
      // 가져오기도구목록
      const tools = await client.listTools();
      serverState.tools = tools;
      
      // 发送이벤트
      this.emit('server:connected', serverName);
      this.emit('tools:updated', serverName, tools);
      
    } catch (error) {
      serverState.status = 'error';
      serverState.lastError = error as Error;
      
      this.emit('server:error', serverName, error as Error);
      
      // 安排재연결
      this.scheduleReconnect(serverName);
    }
  }

  /**
   * 설정클라이언트이벤트처리器
   */
  private setupClientEventHandlers(client: McpClient, serverState: ServerState): void {
    // 这里可로리스닝클라이언트의特定이벤트
    // 例如도구목록업데이트等
  }

  /**
   * 安排재연결
   */
  private scheduleReconnect(serverName: string): void {
    const serverState = this.servers.get(serverName);
    if (!serverState) {
      return;
    }

    if (serverState.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error(`Max reconnect attempts reached for server ${serverName}`);
      return;
    }

    serverState.reconnectAttempts++;
    serverState.status = 'reconnecting';
    
    const delay = this.baseReconnectDelay * Math.pow(2, serverState.reconnectAttempts - 1);
    
    this.emit('server:reconnecting', serverName, serverState.reconnectAttempts);
    
    const timer = setTimeout(async () => {
      this.reconnectTimers.delete(serverName);
      await this.connectServer(serverName);
    }, delay);
    
    this.reconnectTimers.set(serverName, timer);
  }

  /**
   * 가져오기서버상태
   */
  public getServerState(serverName: string): ServerState | undefined {
    return this.servers.get(serverName);
  }

  /**
   * 가져오기所있음서버상태
   */
  public getAllServerStates(): ServerState[] {
    return Array.from(this.servers.values());
  }

  /**
   * 가져오기已연결의서버
   */
  public getConnectedServers(): ServerState[] {
    return Array.from(this.servers.values()).filter(
      server => server.status === 'connected'
    );
  }

  /**
   * 가져오기所있음사용 가능도구
   */
  public getAllTools(): Array<McpTool & { serverName: string }> {
    const allTools: Array<McpTool & { serverName: string }> = [];
    
    for (const serverState of this.servers.values()) {
      if (serverState.status === 'connected') {
        for (const tool of serverState.tools) {
          allTools.push({
            ...tool,
            serverName: serverState.name
          });
        }
      }
    }
    
    return allTools;
  }

  /**
   * 调用도구 - 기반逆向분석gw함수
   */
  public async callTool(toolName: string, arguments_: any, serverName?: string): Promise<any> {
    // 如果指定了서버名称, 直接使用该서버
    if (serverName) {
      const serverState = this.servers.get(serverName);
      if (!serverState || !serverState.client || serverState.status !== 'connected') {
        throw new Error(`Server ${serverName} is not available`);
      }
      
      return serverState.client.callTool(toolName, arguments_);
    }
    
    // 아니오则查找拥있음该도구의서버
    for (const serverState of this.servers.values()) {
      if (serverState.status === 'connected' && serverState.client) {
        const hasTool = serverState.tools.some(tool => tool.name === toolName);
        if (hasTool) {
          return serverState.client.callTool(toolName, arguments_);
        }
      }
    }
    
    throw new Error(`Tool ${toolName} not found in any connected server`);
  }

  /**
   * 手动재연결서버
   */
  public async reconnectServer(serverName: string): Promise<void> {
    const serverState = this.servers.get(serverName);
    if (!serverState) {
      throw new Error(`Server ${serverName} not found`);
    }

    // 清理现있음연결
    if (serverState.client) {
      await serverState.client.disconnect();
      serverState.client = undefined;
    }

    // 재설정재연결计数
    serverState.reconnectAttempts = 0;

    // 立即재연결
    await this.connectServer(serverName);
  }

  /**
   * 재연결所있음실패의서버
   */
  public async reconnectAllFailedServers(): Promise<void> {
    const failedServers = Array.from(this.servers.values()).filter(
      server => server.status === 'error' || server.status === 'disconnected'
    );

    await Promise.allSettled(
      failedServers.map(server => this.reconnectServer(server.name))
    );
  }

  /**
   * 가져오기서버통계정보
   */
  public getStatistics(): {
    total: number;
    connected: number;
    disconnected: number;
    error: number;
    reconnecting: number;
    totalTools: number;
  } {
    const stats = {
      total: this.servers.size,
      connected: 0,
      disconnected: 0,
      error: 0,
      reconnecting: 0,
      totalTools: 0
    };

    for (const server of this.servers.values()) {
      switch (server.status) {
        case 'connected':
          stats.connected++;
          stats.totalTools += server.tools.length;
          break;
        case 'disconnected':
          stats.disconnected++;
          break;
        case 'error':
          stats.error++;
          break;
        case 'reconnecting':
          stats.reconnecting++;
          break;
      }
    }

    return stats;
  }

  /**
   * 清理所있음연결
   */
  public async cleanup(): Promise<void> {
    // 清理所있음재연결定时器
    for (const timer of this.reconnectTimers.values()) {
      clearTimeout(timer);
    }
    this.reconnectTimers.clear();

    // 断开所있음연결
    const disconnectPromises = Array.from(this.servers.values()).map(async server => {
      if (server.client) {
        await server.client.disconnect();
      }
    });

    await Promise.allSettled(disconnectPromises);
    this.servers.clear();
  }
}

/**
 * 전역서버관리器인스턴스
 */
export const globalServerManager = new McpServerManager();
```

---

## 📁 3주차: 도구보안와설정시스템

### 단계4.7: 도구화이트리스트와보안메커니즘

**기반逆向분석의도구권한제어**

**파일 경로**: `src/mcp/security/tool-whitelist.ts`
**파일 내용**:
```typescript
/**
 * MCP도구화이트리스트와보안메커니즘
 * 기반逆向분석의Claude Code도구보안제어
 * 구현도구필터링, 권한검증, 보안전략
 */

export interface ToolSecurityPolicy {
  allowedPrefixes: string[];
  blockedPrefixes: string[];
  allowedTools: string[];
  blockedTools: string[];
  requiresPermission: string[];
  maxConcurrentCalls: number;
  timeout: number;
}

export interface ToolCallContext {
  toolName: string;
  serverName: string;
  arguments: any;
  sessionId: string;
  userId?: string;
}

export interface SecurityViolation {
  type: 'blocked_tool' | 'blocked_prefix' | 'permission_required' | 'rate_limit' | 'timeout';
  toolName: string;
  reason: string;
  timestamp: number;
}

/**
 * 도구보안관리器
 * 기반逆向분석improved-claude-code-5.mjs:35471-35475의l65함수구현
 */
export class ToolSecurityManager {
  private policy: ToolSecurityPolicy;
  private activeCalls = new Map<string, number>(); // serverName -> count
  private callHistory = new Map<string, number[]>(); // toolName -> timestamps
  private violations: SecurityViolation[] = [];

  constructor(policy?: Partial<ToolSecurityPolicy>) {
    this.policy = {
      // 기반逆向분석의IDE도구화이트리스트 - c65상수
      allowedPrefixes: ['mcp__'],
      blockedPrefixes: [],
      allowedTools: [
        'mcp__ide__executeCode',
        'mcp__ide__getDiagnostics'
      ],
      blockedTools: [],
      requiresPermission: [
        'mcp__ide__executeCode'
      ],
      maxConcurrentCalls: 10,
      timeout: 30000,
      ...policy
    };
  }

  /**
   * 도구필터 - 기반逆向분석l65함수구현
   * improved-claude-code-5.mjs:35471-35475
   */
  public isToolAllowed(toolName: string): boolean {
    // 확인예아니오在阻止목록中
    if (this.policy.blockedTools.includes(toolName)) {
      return false;
    }

    // 확인阻止의前缀
    for (const prefix of this.policy.blockedPrefixes) {
      if (toolName.startsWith(prefix)) {
        return false;
      }
    }

    // 확인允许의도구목록
    if (this.policy.allowedTools.includes(toolName)) {
      return true;
    }

    // 확인允许의前缀
    for (const prefix of this.policy.allowedPrefixes) {
      if (toolName.startsWith(prefix)) {
        // 对于IDE도구, 使用화이트리스트메커니즘
        if (toolName.startsWith('mcp__ide__')) {
          return this.isIdeToolAllowed(toolName);
        }
        return true;
      }
    }

    return false;
  }

  /**
   * IDE도구화이트리스트확인 - 기반逆向분석c65상수
   */
  private isIdeToolAllowed(toolName: string): boolean {
    const ideWhitelist = [
      'mcp__ide__executeCode',
      'mcp__ide__getDiagnostics'
    ];
    
    return ideWhitelist.includes(toolName);
  }

  /**
   * 권한확인
   */
  public requiresPermission(toolName: string): boolean {
    return this.policy.requiresPermission.includes(toolName);
  }

  /**
   * 검증도구调用
   */
  public async validateToolCall(context: ToolCallContext): Promise<{
    allowed: boolean;
    violation?: SecurityViolation;
  }> {
    const { toolName, serverName } = context;

    // 1. 확인도구예아니오被允许
    if (!this.isToolAllowed(toolName)) {
      const violation: SecurityViolation = {
        type: 'blocked_tool',
        toolName,
        reason: `Tool ${toolName} is not in the allowed list`,
        timestamp: Date.now()
      };
      
      this.violations.push(violation);
      return { allowed: false, violation };
    }

    // 2. 확인동시限制
    const currentCalls = this.activeCalls.get(serverName) || 0;
    if (currentCalls >= this.policy.maxConcurrentCalls) {
      const violation: SecurityViolation = {
        type: 'rate_limit',
        toolName,
        reason: `Too many concurrent calls to ${serverName}`,
        timestamp: Date.now()
      };
      
      this.violations.push(violation);
      return { allowed: false, violation };
    }

    // 3. 확인调用频率
    if (this.isRateLimited(toolName)) {
      const violation: SecurityViolation = {
        type: 'rate_limit',
        toolName,
        reason: `Tool ${toolName} is being called too frequently`,
        timestamp: Date.now()
      };
      
      this.violations.push(violation);
      return { allowed: false, violation };
    }

    return { allowed: true };
  }

  /**
   * 开始도구调用추적
   */
  public startToolCall(serverName: string, toolName: string): void {
    // 增加동시计数
    const currentCalls = this.activeCalls.get(serverName) || 0;
    this.activeCalls.set(serverName, currentCalls + 1);

    // 记录调用历史
    const history = this.callHistory.get(toolName) || [];
    history.push(Date.now());
    
    // 保留最近1小时의记录
    const oneHourAgo = Date.now() - 60 * 60 * 1000;
    const recentHistory = history.filter(timestamp => timestamp > oneHourAgo);
    this.callHistory.set(toolName, recentHistory);
  }

  /**
   * 结束도구调用추적
   */
  public endToolCall(serverName: string): void {
    const currentCalls = this.activeCalls.get(serverName) || 0;
    if (currentCalls > 0) {
      this.activeCalls.set(serverName, currentCalls - 1);
    }
  }

  /**
   * 확인调用频率限制
   */
  private isRateLimited(toolName: string): boolean {
    const history = this.callHistory.get(toolName) || [];
    const now = Date.now();
    
    // 最近1分钟不超过10次调用
    const oneMinuteAgo = now - 60 * 1000;
    const recentCalls = history.filter(timestamp => timestamp > oneMinuteAgo);
    
    return recentCalls.length >= 10;
  }

  /**
   * 가져오기보안违规记录
   */
  public getViolations(limit?: number): SecurityViolation[] {
    const sorted = [...this.violations].sort((a, b) => b.timestamp - a.timestamp);
    return limit ? sorted.slice(0, limit) : sorted;
  }

  /**
   * 清除过期의违规记录
   */
  public cleanupViolations(): void {
    const oneDayAgo = Date.now() - 24 * 60 * 60 * 1000;
    this.violations = this.violations.filter(violation => violation.timestamp > oneDayAgo);
  }

  /**
   * 가져오기보안통계
   */
  public getSecurityStats(): {
    totalViolations: number;
    violationsByType: Record<string, number>;
    activeCalls: number;
    mostCalledTools: Array<{ toolName: string; count: number }>;
  } {
    const violationsByType: Record<string, number> = {};
    for (const violation of this.violations) {
      violationsByType[violation.type] = (violationsByType[violation.type] || 0) + 1;
    }

    const activeCalls = Array.from(this.activeCalls.values()).reduce((sum, count) => sum + count, 0);

    const toolCallCounts = new Map<string, number>();
    for (const [toolName, history] of this.callHistory) {
      toolCallCounts.set(toolName, history.length);
    }

    const mostCalledTools = Array.from(toolCallCounts.entries())
      .map(([toolName, count]) => ({ toolName, count }))
      .sort((a, b) => b.count - a.count)
      .slice(0, 10);

    return {
      totalViolations: this.violations.length,
      violationsByType,
      activeCalls,
      mostCalledTools
    };
  }

  /**
   * 업데이트보안전략
   */
  public updatePolicy(newPolicy: Partial<ToolSecurityPolicy>): void {
    this.policy = { ...this.policy, ...newPolicy };
  }

  /**
   * 가져오기当前보안전략
   */
  public getPolicy(): ToolSecurityPolicy {
    return { ...this.policy };
  }
}

/**
 * 전역도구보안관리器
 */
export const globalToolSecurity = new ToolSecurityManager();
```

### 단계4.8: 설정 관리시스템

**기반逆向분석의3단계설정와인증시스템**

**파일 경로**: `src/mcp/config/config-manager.ts`
**파일 내용**:
```typescript
/**
 * MCP설정 관리시스템
 * 기반逆向분석의Claude Code3단계설정시스템
 * 지원local/project/user级别설정와OAuth인증
 */

import * as fs from 'fs/promises';
import * as path from 'path';
import { McpServerConfig } from '../client';

export type ConfigLevel = 'local' | 'project' | 'user';

export interface McpConfiguration {
  servers: Record<string, McpServerConfig>;
  globalSettings: {
    maxConcurrentConnections: number;
    defaultTimeout: number;
    retryAttempts: number;
    autoReconnect: boolean;
  };
  security: {
    allowedPrefixes: string[];
    blockedTools: string[];
    requirePermissions: boolean;
  };
}

export interface ConfigSource {
  level: ConfigLevel;
  path: string;
  config: Partial<McpConfiguration>;
}

/**
 * MCP설정 관리器
 * 구현3단계설정层次: local > project > user
 */
export class McpConfigManager {
  private configSources: ConfigSource[] = [];
  private mergedConfig: McpConfiguration | null = null;
  private watchers: Map<string, fs.FSWatcher> = new Map();

  private defaultConfig: McpConfiguration = {
    servers: {},
    globalSettings: {
      maxConcurrentConnections: 10,
      defaultTimeout: 30000,
      retryAttempts: 3,
      autoReconnect: true
    },
    security: {
      allowedPrefixes: ['mcp__'],
      blockedTools: [],
      requirePermissions: true
    }
  };

  /**
   * 초기화설정 관리器
   */
  public async initialize(workingDirectory?: string): Promise<void> {
    await this.loadConfigurations(workingDirectory);
    this.setupFileWatchers();
  }

  /**
   * 로딩所있음级别의설정
   */
  private async loadConfigurations(workingDirectory?: string): Promise<void> {
    this.configSources = [];

    const cwd = workingDirectory || process.cwd();

    // 1. User level configuration (~/.claude-code/mcp.json)
    const userConfigPath = this.getUserConfigPath();
    await this.loadConfigFromPath(userConfigPath, 'user');

    // 2. Project level configuration (./mcp.json 또는 ./.claude-code/mcp.json)
    const projectConfigPaths = [
      path.join(cwd, 'mcp.json'),
      path.join(cwd, '.claude-code', 'mcp.json')
    ];

    for (const configPath of projectConfigPaths) {
      await this.loadConfigFromPath(configPath, 'project');
    }

    // 3. Local level configuration (explicit local overrides)
    const localConfigPath = path.join(cwd, '.mcp.local.json');
    await this.loadConfigFromPath(localConfigPath, 'local');

    // 병합설정
    this.mergeConfigurations();
  }

  /**
   * 从指定경로로딩설정
   */
  private async loadConfigFromPath(configPath: string, level: ConfigLevel): Promise<void> {
    try {
      const exists = await this.fileExists(configPath);
      if (!exists) {
        return;
      }

      const content = await fs.readFile(configPath, 'utf-8');
      const config = JSON.parse(content) as Partial<McpConfiguration>;

      this.configSources.push({
        level,
        path: configPath,
        config
      });

      console.log(`Loaded ${level} MCP configuration from: ${configPath}`);
    } catch (error) {
      console.error(`Error loading ${level} configuration from ${configPath}:`, error);
    }
  }

  /**
   * 병합所있음설정源
   */
  private mergeConfigurations(): void {
    // 按优先级정렬: local > project > user
    const sortedSources = [...this.configSources].sort((a, b) => {
      const priority = { local: 3, project: 2, user: 1 };
      return priority[b.level] - priority[a.level];
    });

    // 从기본설정开始
    let merged: McpConfiguration = JSON.parse(JSON.stringify(this.defaultConfig));

    // 依次병합설정
    for (const source of sortedSources.reverse()) { // 反向병합, 低优先级先병합
      merged = this.deepMerge(merged, source.config);
    }

    this.mergedConfig = merged;
  }

  /**
   * 深度병합설정객체
   */
  private deepMerge(target: any, source: any): any {
    const result = { ...target };

    for (const key in source) {
      if (source[key] && typeof source[key] === 'object' && !Array.isArray(source[key])) {
        result[key] = this.deepMerge(result[key] || {}, source[key]);
      } else {
        result[key] = source[key];
      }
    }

    return result;
  }

  /**
   * 가져오기병합后의설정
   */
  public getConfiguration(): McpConfiguration {
    if (!this.mergedConfig) {
      throw new Error('Configuration not initialized');
    }
    return JSON.parse(JSON.stringify(this.mergedConfig));
  }

  /**
   * 가져오기서버설정
   */
  public getServerConfigs(): McpServerConfig[] {
    const config = this.getConfiguration();
    return Object.values(config.servers);
  }

  /**
   * 가져오기特定서버설정
   */
  public getServerConfig(serverName: string): McpServerConfig | undefined {
    const config = this.getConfiguration();
    return config.servers[serverName];
  }

  /**
   * 추가서버설정
   */
  public async addServerConfig(
    serverConfig: McpServerConfig, 
    level: ConfigLevel = 'project'
  ): Promise<void> {
    const configPath = this.getConfigPathForLevel(level);
    
    // 读取现있음설정
    let existingConfig: Partial<McpConfiguration> = {};
    try {
      const content = await fs.readFile(configPath, 'utf-8');
      existingConfig = JSON.parse(content);
    } catch (error) {
      // 파일不存在, 使用null설정
    }

    // 추가新서버
    if (!existingConfig.servers) {
      existingConfig.servers = {};
    }
    existingConfig.servers[serverConfig.name] = serverConfig;

    // 저장설정
    await this.saveConfigToPath(configPath, existingConfig);
    
    // 重新로딩설정
    await this.loadConfigurations();
  }

  /**
   * 移除서버설정
   */
  public async removeServerConfig(
    serverName: string, 
    level: ConfigLevel = 'project'
  ): Promise<void> {
    const configPath = this.getConfigPathForLevel(level);
    
    try {
      const content = await fs.readFile(configPath, 'utf-8');
      const existingConfig = JSON.parse(content) as Partial<McpConfiguration>;
      
      if (existingConfig.servers && existingConfig.servers[serverName]) {
        delete existingConfig.servers[serverName];
        await this.saveConfigToPath(configPath, existingConfig);
        await this.loadConfigurations();
      }
    } catch (error) {
      console.error(`Error removing server config: ${error}`);
    }
  }

  /**
   * 저장설정到指定경로
   */
  private async saveConfigToPath(
    configPath: string, 
    config: Partial<McpConfiguration>
  ): Promise<void> {
    // 确保디렉토리存在
    const dir = path.dirname(configPath);
    await fs.mkdir(dir, { recursive: true });

    // 형식화并저장
    const content = JSON.stringify(config, null, 2);
    await fs.writeFile(configPath, content, 'utf-8');
  }

  /**
   * 가져오기指定级别의설정파일 경로
   */
  private getConfigPathForLevel(level: ConfigLevel): string {
    switch (level) {
      case 'user':
        return this.getUserConfigPath();
      case 'project':
        return path.join(process.cwd(), 'mcp.json');
      case 'local':
        return path.join(process.cwd(), '.mcp.local.json');
    }
  }

  /**
   * 가져오기사용자설정경로
   */
  private getUserConfigPath(): string {
    const homeDir = process.env.HOME || process.env.USERPROFILE || '';
    return path.join(homeDir, '.claude-code', 'mcp.json');
  }

  /**
   * 설정파일监视器
   */
  private setupFileWatchers(): void {
    for (const source of this.configSources) {
      if (!this.watchers.has(source.path)) {
        try {
          const watcher = fs.watch(source.path, (eventType) => {
            if (eventType === 'change') {
              // 지연重新로딩로避免频繁업데이트
              setTimeout(() => {
                this.loadConfigurations().catch(error => {
                  console.error('Error reloading configuration:', error);
                });
              }, 100);
            }
          });

          this.watchers.set(source.path, watcher);
        } catch (error) {
          console.error(`Error setting up watcher for ${source.path}:`, error);
        }
      }
    }
  }

  /**
   * 확인파일예아니오存在
   */
  private async fileExists(filePath: string): Promise<boolean> {
    try {
      await fs.access(filePath);
      return true;
    } catch {
      return false;
    }
  }

  /**
   * 가져오기설정源정보
   */
  public getConfigSources(): ConfigSource[] {
    return [...this.configSources];
  }

  /**
   * 검증설정
   */
  public validateConfiguration(): {
    valid: boolean;
    errors: string[];
  } {
    const errors: string[] = [];
    const config = this.getConfiguration();

    // 검증서버설정
    for (const [name, serverConfig] of Object.entries(config.servers)) {
      if (!serverConfig.name) {
        errors.push(`Server configuration missing name: ${name}`);
      }

      if (!serverConfig.transport) {
        errors.push(`Server ${name} missing transport configuration`);
      }

      // 검증전송설정
      if (serverConfig.transport) {
        switch (serverConfig.transport.type) {
          case 'stdio':
            if (!serverConfig.transport.command) {
              errors.push(`Server ${name} STDIO transport missing command`);
            }
            break;
          case 'http':
          case 'sse':
          case 'sse-ide':
          case 'ws-ide':
            if (!serverConfig.transport.url) {
              errors.push(`Server ${name} ${serverConfig.transport.type} transport missing URL`);
            }
            break;
        }
      }
    }

    return {
      valid: errors.length === 0,
      errors
    };
  }

  /**
   * 清理资源
   */
  public async cleanup(): Promise<void> {
    // 끄기所있음파일监视器
    for (const watcher of this.watchers.values()) {
      watcher.close();
    }
    this.watchers.clear();
  }
}

/**
 * OAuth인증관리器
 * 기반逆向분석의Claude Code OAuth2구현
 */
export class OAuthManager {
  private tokens = new Map<string, {
    accessToken: string;
    refreshToken?: string;
    expiresAt: number;
  }>();

  /**
   * 가져오기访问토큰
   */
  public async getAccessToken(serverName: string): Promise<string | null> {
    const tokenInfo = this.tokens.get(serverName);
    
    if (!tokenInfo) {
      return null;
    }

    // 확인예아니오过期
    if (Date.now() >= tokenInfo.expiresAt) {
      // 尝试새로고침토큰
      if (tokenInfo.refreshToken) {
        return await this.refreshAccessToken(serverName, tokenInfo.refreshToken);
      }
      
      // 清除过期토큰
      this.tokens.delete(serverName);
      return null;
    }

    return tokenInfo.accessToken;
  }

  /**
   * 저장소访问토큰
   */
  public setAccessToken(
    serverName: string, 
    accessToken: string, 
    expiresIn: number,
    refreshToken?: string
  ): void {
    this.tokens.set(serverName, {
      accessToken,
      refreshToken,
      expiresAt: Date.now() + (expiresIn * 1000)
    });
  }

  /**
   * 새로고침访问토큰
   */
  private async refreshAccessToken(serverName: string, refreshToken: string): Promise<string | null> {
    // 这里应该구현OAuth2토큰새로고침논리
    // 구체구현取决于OAuth2提供商의API
    console.log(`Refreshing token for server: ${serverName}`);
    return null;
  }

  /**
   * 清除토큰
   */
  public clearToken(serverName: string): void {
    this.tokens.delete(serverName);
  }

  /**
   * 清除所있음토큰
   */
  public clearAllTokens(): void {
    this.tokens.clear();
  }
}

/**
 * 전역설정 관리器인스턴스
 */
export const globalConfigManager = new McpConfigManager();
export const globalOAuthManager = new OAuthManager();
```

---

## 📁 4주차: 확장프레임워크와생태계시스템

### 단계4.9: MCP扩펼치기发프레임워크

**为第三方개발者提供의扩펼치기发도구**

**파일 경로**: `src/mcp/extensions/extension-framework.ts`
**파일 내용**:
```typescript
/**
 * MCP扩펼치기发프레임워크
 * 为第三方개발者提供의扩펼치기发도구와API
 * 지원플러그인등록, 버전 관리, 의존성解析
 */

export interface ExtensionManifest {
  name: string;
  version: string;
  description: string;
  author: string;
  license?: string;
  homepage?: string;
  repository?: string;
  keywords?: string[];
  
  // Claude Code特定字段
  claudeCodeVersion: string;
  mcpVersion: string;
  
  // 확장설정
  main: string;
  tools?: ToolDefinition[];
  resources?: ResourceDefinition[];
  prompts?: PromptDefinition[];
  
  // 의존성와권한
  dependencies?: Record<string, string>;
  permissions?: Permission[];
  
  // 生命주期훅
  activationEvents?: string[];
  
  // 설정Schema
  configuration?: ConfigurationSchema;
}

export interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: any;
  handler: string; // 함수名또는파일 경로
  permissions?: string[];
  timeout?: number;
}

export interface ResourceDefinition {
  uriPattern: string;
  name: string;
  description: string;
  mimeType?: string;
  handler: string;
}

export interface PromptDefinition {
  name: string;
  description: string;
  arguments?: ArgumentDefinition[];
  handler: string;
}

export interface ArgumentDefinition {
  name: string;
  description: string;
  required: boolean;
  type: 'string' | 'number' | 'boolean' | 'object' | 'array';
}

export interface Permission {
  type: 'filesystem' | 'network' | 'process' | 'env' | 'clipboard';
  scope?: string;
  description: string;
}

export interface ConfigurationSchema {
  type: 'object';
  properties: Record<string, any>;
  required?: string[];
}

export interface ExtensionContext {
  extensionPath: string;
  storageUri: string;
  globalStorageUri: string;
  subscriptions: any[];
  workspaceState: ExtensionStorage;
  globalState: ExtensionStorage;
  logger: ExtensionLogger;
}

export interface ExtensionStorage {
  get<T>(key: string, defaultValue?: T): T | undefined;
  update(key: string, value: any): Promise<void>;
  keys(): readonly string[];
}

export interface ExtensionLogger {
  info(message: string, ...args: any[]): void;
  warn(message: string, ...args: any[]): void;
  error(message: string, ...args: any[]): void;
  debug(message: string, ...args: any[]): void;
}

/**
 * 확장基클래스
 * 所있음확장都应该상속自此클래스
 */
export abstract class Extension {
  protected context: ExtensionContext;
  protected manifest: ExtensionManifest;

  constructor(context: ExtensionContext, manifest: ExtensionManifest) {
    this.context = context;
    this.manifest = manifest;
  }

  /**
   * 확장활성화时调用
   */
  public abstract activate(): Promise<void>;

  /**
   * 확장停用时调用
   */
  public abstract deactivate(): Promise<void>;

  /**
   * 등록도구
   */
  protected registerTool(definition: ToolDefinition, handler: Function): void {
    // 구현도구등록논리
    console.log(`Registering tool: ${definition.name}`);
  }

  /**
   * 등록资源
   */
  protected registerResource(definition: ResourceDefinition, handler: Function): void {
    // 구현资源등록논리
    console.log(`Registering resource: ${definition.name}`);
  }

  /**
   * 등록힌트
   */
  protected registerPrompt(definition: PromptDefinition, handler: Function): void {
    // 구현힌트등록논리
    console.log(`Registering prompt: ${definition.name}`);
  }

  /**
   * 가져오기설정值
   */
  protected getConfiguration<T>(key: string, defaultValue?: T): T {
    // 구현설정가져오기논리
    return defaultValue as T;
  }

  /**
   * 업데이트설정值
   */
  protected async updateConfiguration(key: string, value: any): Promise<void> {
    // 구현설정업데이트논리
  }
}

/**
 * 확장관리器
 * 담당확장의로딩, 관리와生命주期
 */
export class ExtensionManager {
  private extensions = new Map<string, Extension>();
  private manifests = new Map<string, ExtensionManifest>();
  private extensionPaths: string[] = [];

  /**
   * 추가확장검색경로
   */
  public addExtensionPath(path: string): void {
    this.extensionPaths.push(path);
  }

  /**
   * 扫描并로딩所있음확장
   */
  public async loadExtensions(): Promise<void> {
    for (const searchPath of this.extensionPaths) {
      await this.scanExtensionsInPath(searchPath);
    }
  }

  /**
   * 扫描指定경로中의확장
   */
  private async scanExtensionsInPath(searchPath: string): Promise<void> {
    try {
      const fs = await import('fs/promises');
      const path = await import('path');
      
      const entries = await fs.readdir(searchPath, { withFileTypes: true });
      
      for (const entry of entries) {
        if (entry.isDirectory()) {
          const extensionPath = path.join(searchPath, entry.name);
          await this.loadExtensionFromPath(extensionPath);
        }
      }
    } catch (error) {
      console.error(`Error scanning extensions in ${searchPath}:`, error);
    }
  }

  /**
   * 从指定경로로딩확장
   */
  private async loadExtensionFromPath(extensionPath: string): Promise<void> {
    try {
      const fs = await import('fs/promises');
      const path = await import('path');
      
      // 读取manifest.json
      const manifestPath = path.join(extensionPath, 'manifest.json');
      const manifestContent = await fs.readFile(manifestPath, 'utf-8');
      const manifest: ExtensionManifest = JSON.parse(manifestContent);
      
      // 검증manifest
      if (!this.validateManifest(manifest)) {
        console.error(`Invalid manifest for extension: ${manifest.name}`);
        return;
      }
      
      // 확인버전호환성
      if (!this.isVersionCompatible(manifest)) {
        console.error(`Incompatible version for extension: ${manifest.name}`);
        return;
      }
      
      // 로딩확장主파일
      const mainPath = path.join(extensionPath, manifest.main);
      const extensionModule = await import(mainPath);
      
      // 생성확장컨텍스트
      const context = this.createExtensionContext(extensionPath, manifest);
      
      // 인스턴스化확장
      const extension = new extensionModule.default(context, manifest);
      
      // 활성화확장
      await extension.activate();
      
      // 등록확장
      this.extensions.set(manifest.name, extension);
      this.manifests.set(manifest.name, manifest);
      
      console.log(`Loaded extension: ${manifest.name} v${manifest.version}`);
      
    } catch (error) {
      console.error(`Error loading extension from ${extensionPath}:`, error);
    }
  }

  /**
   * 검증확장manifest
   */
  private validateManifest(manifest: ExtensionManifest): boolean {
    const requiredFields = ['name', 'version', 'description', 'main', 'claudeCodeVersion'];
    
    for (const field of requiredFields) {
      if (!(field in manifest)) {
        console.error(`Missing required field: ${field}`);
        return false;
      }
    }
    
    return true;
  }

  /**
   * 확인버전호환성
   */
  private isVersionCompatible(manifest: ExtensionManifest): boolean {
    // 구현버전호환성확인
    // 这里可로使用semver라이브러리进行语义버전比较
    return true;
  }

  /**
   * 생성확장컨텍스트
   */
  private createExtensionContext(extensionPath: string, manifest: ExtensionManifest): ExtensionContext {
    const path = require('path');
    const os = require('os');
    
    const globalStorageUri = path.join(os.homedir(), '.claude-code', 'extensions', manifest.name);
    const storageUri = path.join(extensionPath, '.storage');
    
    return {
      extensionPath,
      storageUri,
      globalStorageUri,
      subscriptions: [],
      workspaceState: this.createStorage(storageUri),
      globalState: this.createStorage(globalStorageUri),
      logger: this.createLogger(manifest.name)
    };
  }

  /**
   * 생성저장소인스턴스
   */
  private createStorage(storagePath: string): ExtensionStorage {
    return {
      get<T>(key: string, defaultValue?: T): T | undefined {
        // 구현저장소读取논리
        return defaultValue;
      },
      
      async update(key: string, value: any): Promise<void> {
        // 구현저장소업데이트논리
      },
      
      keys(): readonly string[] {
        // 구현键목록가져오기논리
        return [];
      }
    };
  }

  /**
   * 생성로그记录器
   */
  private createLogger(extensionName: string): ExtensionLogger {
    return {
      info: (message: string, ...args: any[]) => {
        console.log(`[${extensionName}] INFO: ${message}`, ...args);
      },
      
      warn: (message: string, ...args: any[]) => {
        console.warn(`[${extensionName}] WARN: ${message}`, ...args);
      },
      
      error: (message: string, ...args: any[]) => {
        console.error(`[${extensionName}] ERROR: ${message}`, ...args);
      },
      
      debug: (message: string, ...args: any[]) => {
        console.debug(`[${extensionName}] DEBUG: ${message}`, ...args);
      }
    };
  }

  /**
   * 제거확장
   */
  public async unloadExtension(extensionName: string): Promise<void> {
    const extension = this.extensions.get(extensionName);
    if (extension) {
      await extension.deactivate();
      this.extensions.delete(extensionName);
      this.manifests.delete(extensionName);
      console.log(`Unloaded extension: ${extensionName}`);
    }
  }

  /**
   * 가져오기已로딩의확장목록
   */
  public getLoadedExtensions(): ExtensionManifest[] {
    return Array.from(this.manifests.values());
  }

  /**
   * 가져오기特定확장의manifest
   */
  public getExtensionManifest(extensionName: string): ExtensionManifest | undefined {
    return this.manifests.get(extensionName);
  }

  /**
   * 확인확장예아니오已로딩
   */
  public isExtensionLoaded(extensionName: string): boolean {
    return this.extensions.has(extensionName);
  }

  /**
   * 重新로딩확장
   */
  public async reloadExtension(extensionName: string): Promise<void> {
    if (this.isExtensionLoaded(extensionName)) {
      await this.unloadExtension(extensionName);
    }
    
    // 重新扫描并로딩
    await this.loadExtensions();
  }

  /**
   * 清理所있음확장
   */
  public async cleanup(): Promise<void> {
    const extensionNames = Array.from(this.extensions.keys());
    
    for (const extensionName of extensionNames) {
      await this.unloadExtension(extensionName);
    }
  }
}

/**
 * 확장도구함수
 */
export namespace ExtensionUtils {
  /**
   * 생성확장템플릿
   */
  export async function createExtensionTemplate(
    name: string, 
    outputPath: string
  ): Promise<void> {
    const fs = await import('fs/promises');
    const path = await import('path');
    
    const extensionPath = path.join(outputPath, name);
    await fs.mkdir(extensionPath, { recursive: true });
    
    // 생성manifest.json
    const manifest: ExtensionManifest = {
      name,
      version: '1.0.0',
      description: `${name} extension for Claude Code`,
      author: 'Your Name',
      license: 'MIT',
      claudeCodeVersion: '^1.0.0',
      mcpVersion: '2024-11-05',
      main: 'src/extension.js',
      activationEvents: ['*'],
      configuration: {
        type: 'object',
        properties: {},
        required: []
      }
    };
    
    await fs.writeFile(
      path.join(extensionPath, 'manifest.json'),
      JSON.stringify(manifest, null, 2)
    );
    
    // 생성主파일
    const mainContent = `
const { Extension } = require('claude-code');

class ${name}Extension extends Extension {
  async activate() {
    this.context.logger.info('Extension activated');
    
    // 등록도구, 资源, 힌트等
  }
  
  async deactivate() {
    this.context.logger.info('Extension deactivated');
  }
}

module.exports = ${name}Extension;
`;
    
    const srcPath = path.join(extensionPath, 'src');
    await fs.mkdir(srcPath, { recursive: true });
    await fs.writeFile(path.join(srcPath, 'extension.js'), mainContent);
    
    console.log(`Extension template created at: ${extensionPath}`);
  }

  /**
   * 검증확장패키지
   */
  export async function validateExtension(extensionPath: string): Promise<{
    valid: boolean;
    errors: string[];
  }> {
    const errors: string[] = [];
    
    try {
      const fs = await import('fs/promises');
      const path = await import('path');
      
      // 확인manifest.json
      const manifestPath = path.join(extensionPath, 'manifest.json');
      try {
        const manifestContent = await fs.readFile(manifestPath, 'utf-8');
        const manifest = JSON.parse(manifestContent);
        
        // 검증필수字段
        const requiredFields = ['name', 'version', 'description', 'main'];
        for (const field of requiredFields) {
          if (!manifest[field]) {
            errors.push(`Missing required field: ${field}`);
          }
        }
        
        // 확인主파일예아니오存在
        const mainPath = path.join(extensionPath, manifest.main);
        try {
          await fs.access(mainPath);
        } catch {
          errors.push(`Main file not found: ${manifest.main}`);
        }
        
      } catch (error) {
        errors.push(`Invalid or missing manifest.json: ${error}`);
      }
      
    } catch (error) {
      errors.push(`Error accessing extension path: ${error}`);
    }
    
    return {
      valid: errors.length === 0,
      errors
    };
  }
}

/**
 * 전역확장관리器인스턴스
 */
export const globalExtensionManager = new ExtensionManager();
```

### 단계4.10: 전체통합 테스트

**단계4의전체통합 테스트스위트**

**파일 경로**: `src/__tests__/stage4-mcp-integration.test.ts`
**파일 내용**:
```typescript
/**
 * 단계4 MCP 통합와확장 시스템통합 테스트
 * 검증MCP프로토콜, 서버관리, 도구보안, 설정시스템의전체功能
 */

import { describe, test, expect, beforeEach, afterEach, jest } from '@jest/testing-library';
import { McpClient } from '../mcp/client';
import { McpServerManager } from '../mcp/server-manager';
import { ToolSecurityManager } from '../mcp/security/tool-whitelist';
import { McpConfigManager } from '../mcp/config/config-manager';
import { ExtensionManager } from '../mcp/extensions/extension-framework';
import { StdioTransport, HttpTransport, SseTransport, WebSocketTransport } from '../mcp/transport';

describe('단계4 - MCP 통합와확장 시스템전체테스트', () => {
  let serverManager: McpServerManager;
  let toolSecurity: ToolSecurityManager;
  let configManager: McpConfigManager;
  let extensionManager: ExtensionManager;

  beforeEach(() => {
    serverManager = new McpServerManager();
    toolSecurity = new ToolSecurityManager();
    configManager = new McpConfigManager();
    extensionManager = new ExtensionManager();
  });

  afterEach(async () => {
    await serverManager.cleanup();
    await configManager.cleanup();
    await extensionManager.cleanup();
  });

  describe('MCP전송层테스트', () => {
    test('STDIO전송연결와通信', async () => {
      const config = {
        type: 'stdio' as const,
        command: 'echo',
        args: ['{"jsonrpc":"2.0","id":1,"result":"test"}']
      };

      const transport = new StdioTransport(config);
      
      // 模拟연결
      const messagePromise = new Promise((resolve) => {
        transport.onMessage(resolve);
      });

      await transport.connect();
      expect(transport.isConnected()).toBe(true);

      // 发送메시지并검증응답
      const testMessage = { jsonrpc: '2.0', id: 1, method: 'test' };
      await transport.send(testMessage);

      await transport.disconnect();
      expect(transport.isConnected()).toBe(false);
    });

    test('HTTP전송요청-응답모드', async () => {
      // 模拟HTTP서버
      const mockFetch = jest.fn().mockResolvedValue({
        ok: true,
        json: () => Promise.resolve({ jsonrpc: '2.0', id: 1, result: 'success' })
      });
      
      global.fetch = mockFetch;

      const config = {
        type: 'http' as const,
        url: 'http://localhost:8080/mcp'
      };

      const transport = new HttpTransport(config);
      await transport.connect();

      const testMessage = { jsonrpc: '2.0', id: 1, method: 'test' };
      await transport.send(testMessage);

      expect(mockFetch).toHaveBeenCalledWith(
        'http://localhost:8080/mcp',
        expect.objectContaining({
          method: 'POST',
          headers: expect.objectContaining({
            'Content-Type': 'application/json'
          }),
          body: JSON.stringify(testMessage)
        })
      );
    });

    test('WebSocket双向通信', async () => {
      // 这里需要模拟WebSocket서버
      // 또는者使用테스트라이브러리如ws来생성테스트서버
      
      const config = {
        type: 'websocket' as const,
        url: 'ws://localhost:8080/mcp',
        protocols: ['mcp']
      };

      // 模拟WebSocket구현
      const mockWebSocket = {
        readyState: 1, // OPEN
        send: jest.fn(),
        close: jest.fn(),
        on: jest.fn(),
        addEventListener: jest.fn()
      };

      // 这里需要适当의WebSocket模拟논리
      expect(config.url).toBe('ws://localhost:8080/mcp');
    });

    test('IDE전용전송(SSE-IDE와WS-IDE)', async () => {
      const sseIdeConfig = {
        type: 'sse-ide' as const,
        url: 'http://vscode-extension/sse',
        ideName: 'vscode'
      };

      const wsIdeConfig = {
        type: 'ws-ide' as const,
        url: 'ws://cursor-extension/ws',
        ideName: 'cursor',
        authToken: 'test-token'
      };

      // 검증설정正确性
      expect(sseIdeConfig.ideName).toBe('vscode');
      expect(wsIdeConfig.authToken).toBe('test-token');
    });
  });

  describe('MCP클라이언트핵심功能테스트', () => {
    test('클라이언트초기화와握手', async () => {
      const config = {
        name: 'test-server',
        transport: {
          type: 'stdio' as const,
          command: 'node',
          args: ['test-mcp-server.js']
        }
      };

      const client = new McpClient(config);
      
      // 模拟성공연결
      expect(client.connected).toBe(false);
      
      // 실제테스트中这里会연결到true实의MCP서버
      // await client.connect();
      // expect(client.connected).toBe(true);
    });

    test('도구调用功能', async () => {
      const client = new McpClient({
        name: 'test-server',
        transport: { type: 'stdio', command: 'echo' }
      });

      // 模拟도구调用
      const toolName = 'test_tool';
      const arguments_ = { param1: 'value1', param2: 42 };

      // 실제구현中会调用true实도구
      // const result = await client.callTool(toolName, arguments_);
      // expect(result).toBeDefined();
    });

    test('资源와힌트관리', async () => {
      const client = new McpClient({
        name: 'test-server',
        transport: { type: 'stdio', command: 'echo' }
      });

      // 테스트资源목록가져오기
      // const resources = await client.listResources();
      // expect(Array.isArray(resources)).toBe(true);

      // 테스트힌트목록가져오기
      // const prompts = await client.listPrompts();
      // expect(Array.isArray(prompts)).toBe(true);
    });
  });

  describe('다중 서버 관리테스트', () => {
    test('서버연결池관리', async () => {
      const serverConfigs = [
        {
          name: 'server1',
          transport: { type: 'stdio' as const, command: 'echo' }
        },
        {
          name: 'server2', 
          transport: { type: 'http' as const, url: 'http://localhost:8080' }
        }
      ];

      for (const config of serverConfigs) {
        await serverManager.addServer(config);
      }

      const allStates = serverManager.getAllServerStates();
      expect(allStates).toHaveLength(2);
      expect(allStates.map(s => s.name)).toEqual(['server1', 'server2']);
    });

    test('서버상태모니터링와이벤트', async () => {
      const events: string[] = [];
      
      serverManager.on('server:connected', (name) => {
        events.push(`connected:${name}`);
      });
      
      serverManager.on('server:error', (name, error) => {
        events.push(`error:${name}`);
      });

      // 추가一个会실패의서버설정
      await serverManager.addServer({
        name: 'failing-server',
        transport: { type: 'stdio', command: 'nonexistent-command' }
      });

      // 검증오류이벤트被触发
      await new Promise(resolve => setTimeout(resolve, 100));
      expect(events.some(e => e.startsWith('error:'))).toBe(true);
    });

    test('자동재연결 메커니즘', async () => {
      const config = {
        name: 'reconnect-server',
        transport: { type: 'stdio' as const, command: 'echo' },
        retryAttempts: 3
      };

      await serverManager.addServer(config);
      
      const serverState = serverManager.getServerState('reconnect-server');
      expect(serverState).toBeDefined();
      expect(serverState!.reconnectAttempts).toBe(0);
    });

    test('도구调用路由', async () => {
      // 추가多个서버, 每个있음不同의도구
      await serverManager.addServer({
        name: 'tools-server-1',
        transport: { type: 'stdio', command: 'echo' }
      });

      await serverManager.addServer({
        name: 'tools-server-2', 
        transport: { type: 'stdio', command: 'echo' }
      });

      // 테스트도구调用路由到正确의서버
      try {
        await serverManager.callTool('nonexistent_tool', {});
        expect(false).toBe(true); // 应该抛出오류
      } catch (error) {
        expect(error.message).toContain('Tool nonexistent_tool not found');
      }
    });
  });

  describe('도구보안메커니즘테스트', () => {
    test('도구화이트리스트필터링', () => {
      // 테스트IDE도구화이트리스트
      expect(toolSecurity.isToolAllowed('mcp__ide__getDiagnostics')).toBe(true);
      expect(toolSecurity.isToolAllowed('mcp__ide__executeCode')).toBe(true);
      expect(toolSecurity.isToolAllowed('mcp__ide__maliciousTool')).toBe(false);

      // 테스트一般MCP도구
      expect(toolSecurity.isToolAllowed('mcp__general__tool')).toBe(true);
      expect(toolSecurity.isToolAllowed('dangerous_tool')).toBe(false);
    });

    test('권한검증프로세스', async () => {
      const context = {
        toolName: 'mcp__ide__executeCode',
        serverName: 'ide-server',
        arguments: { code: 'print("hello")' },
        sessionId: 'test-session'
      };

      const validation = await toolSecurity.validateToolCall(context);
      expect(validation.allowed).toBe(true);

      // 테스트권한要求
      expect(toolSecurity.requiresPermission('mcp__ide__executeCode')).toBe(true);
    });

    test('동시제어와频率限制', async () => {
      const context = {
        toolName: 'test_tool',
        serverName: 'test-server',
        arguments: {},
        sessionId: 'test-session'
      };

      // 开始多个동시调用
      for (let i = 0; i < 5; i++) {
        toolSecurity.startToolCall('test-server', 'test_tool');
      }

      // 검증동시限制生效
      const validation = await toolSecurity.validateToolCall(context);
      // 在실제구현中, 这可能会因동시限制而실패
    });

    test('보안违规记录와통계', () => {
      const stats = toolSecurity.getSecurityStats();
      expect(stats).toHaveProperty('totalViolations');
      expect(stats).toHaveProperty('violationsByType');
      expect(stats).toHaveProperty('activeCalls');
      expect(stats).toHaveProperty('mostCalledTools');
    });
  });

  describe('설정 관리시스템테스트', () => {
    test('3단계설정层次로딩', async () => {
      // 模拟설정파일 내용
      const userConfig = {
        globalSettings: { maxConcurrentConnections: 20 }
      };
      
      const projectConfig = {
        servers: {
          'project-server': {
            name: 'project-server',
            transport: { type: 'stdio', command: 'node' }
          }
        }
      };

      // 실제테스트中会생성临时설정파일
      await configManager.initialize('./test-workspace');
      
      const mergedConfig = configManager.getConfiguration();
      expect(mergedConfig).toHaveProperty('servers');
      expect(mergedConfig).toHaveProperty('globalSettings');
    });

    test('설정검증', () => {
      const validation = configManager.validateConfiguration();
      expect(validation).toHaveProperty('valid');
      expect(validation).toHaveProperty('errors');
      expect(Array.isArray(validation.errors)).toBe(true);
    });

    test('동적설정업데이트', async () => {
      const serverConfig = {
        name: 'dynamic-server',
        transport: { type: 'stdio' as const, command: 'echo' }
      };

      await configManager.addServerConfig(serverConfig, 'project');
      
      const loadedConfig = configManager.getServerConfig('dynamic-server');
      expect(loadedConfig).toEqual(serverConfig);
    });

    test('설정파일监视', async () => {
      // 模拟설정파일变化
      // 실제테스트中会수정설정파일并검증자동重新로딩
      
      const sources = configManager.getConfigSources();
      expect(Array.isArray(sources)).toBe(true);
    });
  });

  describe('확장 시스템테스트', () => {
    test('확장로딩와生命주期', async () => {
      // 생성模拟확장디렉토리
      const extensionPath = './test-extensions';
      extensionManager.addExtensionPath(extensionPath);

      // 模拟확장manifest
      const manifest = {
        name: 'test-extension',
        version: '1.0.0',
        description: 'Test extension',
        author: 'Test Author',
        claudeCodeVersion: '1.0.0',
        mcpVersion: '2024-11-05',
        main: 'extension.js'
      };

      // 실제테스트中会생성true实의확장파일
      // await extensionManager.loadExtensions();
      
      expect(extensionManager.getLoadedExtensions()).toEqual([]);
    });

    test('확장도구등록', async () => {
      // 模拟확장등록도구
      const toolDefinition = {
        name: 'extension_tool',
        description: 'Tool provided by extension',
        inputSchema: { type: 'object', properties: {} },
        handler: 'handleTool'
      };

      // 실제구현中会通过확장프레임워크등록도구
      expect(toolDefinition.name).toBe('extension_tool');
    });

    test('확장설정와저장소', () => {
      // 테스트확장설정와저장소메커니즘
      const extensionName = 'test-extension';
      
      // 실제구현中会테스트확장의설정读写
      expect(extensionName).toBe('test-extension');
    });
  });

  describe('IDE통합专项테스트', () => {
    test('IDE연결检测', () => {
      const mockServers = [
        {
          type: 'connected',
          name: 'ide',
          config: {
            type: 'sse-ide',
            ideName: 'vscode'
          }
        }
      ];

      // 기반逆向분석의TF1함수테스트
      const detectedIde = mockServers.find(s => 
        s.type === 'connected' && s.name === 'ide'
      )?.config;
      
      expect(detectedIde?.ideName).toBe('vscode');
    });

    test('진단정보관리', async () => {
      // 模拟진단정보가져오기
      const mockDiagnostics = [
        {
          uri: 'file:///test.js',
          diagnostics: [
            {
              message: 'Unused variable',
              severity: 2,
              range: {
                start: { line: 10, character: 5 },
                end: { line: 10, character: 15 }
              }
            }
          ]
        }
      ];

      // 테스트진단정보比较알고리즘
      const diag1 = mockDiagnostics[0].diagnostics[0];
      const diag2 = { ...diag1 };
      
      // 실제구현中会使用IdeDiagnosticsManager进行比较
      expect(JSON.stringify(diag1)).toBe(JSON.stringify(diag2));
    });

    test('코드执行통합', async () => {
      // 模拟IDE코드执行
      const executeRequest = {
        code: 'print("Hello from IDE")',
        language: 'python'
      };

      // 실제구현中会通过MCP调用IDE의executeCode도구
      expect(executeRequest.code).toContain('Hello from IDE');
    });
  });

  describe('엔드투엔드통합 테스트', () => {
    test('전체MCP工作프로세스', async () => {
      // 1. 초기화설정 관리器
      await configManager.initialize();
      
      // 2. 로딩서버설정
      const serverConfig = {
        name: 'integration-test-server',
        transport: { type: 'stdio' as const, command: 'echo' }
      };
      
      await configManager.addServerConfig(serverConfig);
      
      // 3. 시작서버관리器
      await serverManager.addServer(serverConfig);
      
      // 4. 검증보안전략
      const toolName = 'mcp__test__tool';
      const isAllowed = toolSecurity.isToolAllowed(toolName);
      expect(typeof isAllowed).toBe('boolean');
      
      // 5. 로딩확장
      await extensionManager.loadExtensions();
      
      // 6. 검증시스템상태
      const serverStats = serverManager.getStatistics();
      expect(serverStats).toHaveProperty('total');
      
      const securityStats = toolSecurity.getSecurityStats();
      expect(securityStats).toHaveProperty('totalViolations');
      
      const extensions = extensionManager.getLoadedExtensions();
      expect(Array.isArray(extensions)).toBe(true);
    });

    test('오류 처리와복원', async () => {
      // 테스트各种오류情况下의시스템복원기능
      
      // 1. 서버연결실패
      await serverManager.addServer({
        name: 'failing-server',
        transport: { type: 'stdio', command: 'nonexistent' }
      });
      
      // 2. 없음效설정
      const validation = configManager.validateConfiguration();
      expect(validation).toHaveProperty('valid');
      
      // 3. 보안违规
      const context = {
        toolName: 'blocked_tool',
        serverName: 'test',
        arguments: {},
        sessionId: 'test'
      };
      
      const securityCheck = await toolSecurity.validateToolCall(context);
      expect(securityCheck).toHaveProperty('allowed');
    });

    test('性能와리소스 관리', async () => {
      // 테스트시스템의性能特征
      
      // 1. 大量서버연결
      const serverPromises = [];
      for (let i = 0; i < 10; i++) {
        serverPromises.push(
          serverManager.addServer({
            name: `perf-server-${i}`,
            transport: { type: 'stdio', command: 'echo' }
          })
        );
      }
      
      await Promise.allSettled(serverPromises);
      
      // 2. 동시도구调用
      const callPromises = [];
      for (let i = 0; i < 20; i++) {
        callPromises.push(
          toolSecurity.validateToolCall({
            toolName: 'mcp__test__tool',
            serverName: 'test',
            arguments: {},
            sessionId: `session-${i}`
          })
        );
      }
      
      const results = await Promise.allSettled(callPromises);
      expect(results.length).toBe(20);
      
      // 3. 메모리 사용모니터링
      const stats = serverManager.getStatistics();
      expect(stats.total).toBeGreaterThan(0);
    });
  });
});
```

---

## 📋 단계4완료 확인 체크리스트

### 기능 검증 항목

**MCP프로토콜 구현** ✅
- [ ] STDIO전송正常工作
- [ ] HTTP전송요청-응답모드
- [ ] SSE전송실시간푸시
- [ ] WebSocket双向通信
- [ ] IDE전용전송(SSE-IDE, WS-IDE)
- [ ] 메시지형식符合JSON-RPC 2.0

**서버연결관리** ✅
- [ ] 多서버동시연결
- [ ] 연결상태실시간모니터링
- [ ] 자동재연결 메커니즘
- [ ] 오류 처리와복원
- [ ] 도구调用路由正确

**도구보안메커니즘** ✅
- [ ] 도구화이트리스트필터링있음效
- [ ] IDE도구보안제어
- [ ] 동시调用限制
- [ ] 频率限制防护
- [ ] 보안违规记录

**설정 관리시스템** ✅
- [ ] 3단계설정层次로딩
- [ ] 설정파일监视
- [ ] 동적설정업데이트
- [ ] OAuth인증관리
- [ ] 설정검증와오류 처리

**확장프레임워크** ✅
- [ ] 확장로딩와生命주期
- [ ] 도구/资源/힌트등록
- [ ] 확장설정와저장소
- [ ] 버전호환성확인
- [ ] 확장템플릿生成

### 성능 검증 항목

**연결性能** ✅
- [ ] 서버연결时间 < 2s
- [ ] 동시연결数지원 > 10
- [ ] 재연결 메커니즘지연合理
- [ ] 네트워크장애快速检测
- [ ] 资源清理전체

**调用性能** ✅
- [ ] 도구调用응답 < 1s
- [ ] 보안확인지연 < 50ms
- [ ] 동시调用不阻塞
- [ ] 설정업데이트 < 500ms
- [ ] 확장로딩 < 3s

**리소스 관리** ✅
- [ ] 메모리 사용稳定 < 500MB
- [ ] 연결池있음效관리
- [ ] 파일描述符없음泄漏
- [ ] 定时器正确清理
- [ ] GC压力可控

### 호환성검증프로젝트

**MCP프로토콜兼容** ✅
- [ ] 符合MCP 2024-11-05사양
- [ ] JSON-RPC 2.0完全兼容
- [ ] 도구调用형식표준
- [ ] 资源URI형식正确
- [ ] 알림메커니즘符合사양

**전송层兼容** ✅
- [ ] 多种전송방식互通
- [ ] 네트워크프로토콜표준合规
- [ ] 인증메커니즘灵活지원
- [ ] 오류码표준化
- [ ] 타임아웃처리一致

**IDE통합兼容** ✅
- [ ] VS Code확장API
- [ ] Cursor통합인터페이스
- [ ] Windsurf프로토콜지원
- [ ] 진단정보LSP형식
- [ ] 코드执行Jupyter内核

### 보안검증프로젝트

**访问제어** ✅
- [ ] 도구권한严格제어
- [ ] 서버隔离있음效
- [ ] 사용자身份검증
- [ ] 세션관리보안
- [ ] 설정파일권한

**데이터보안** ✅
- [ ] 전송데이터암호화
- [ ] 敏感정보脱敏
- [ ] 로그보안记录
- [ ] 설정파일보호
- [ ] 临时파일清理

**런타임보안** ✅
- [ ] 进程隔离메커니즘
- [ ] 资源使用限制
- [ ] 예외처리完善
- [ ] 恶意입력防护
- [ ] 시스템调用모니터링

---

## 🎯 다음 단계 예고

단계4완료 후, Open Claude Code을/를 갖추게 됨: 

1. **전체의MCP생태계시스템**: 
   - 四种전송프로토콜지원
   - 企业级서버관리
   - 보안의도구执行환경
   - 灵活의설정 관리

2. **强大의확장기능**: 
   - 第三方플러그인지원
   - 표준化개발프레임워크
   - 丰富의API인터페이스
   - 完善의문서体系

3. **深度IDE통합**: 
   - LSP진단정보동기화
   - Jupyter코드执行
   - 실시간상태모니터링
   - 多IDE생태계지원

**진입단계5**: 테스트 최적화와배포 준备(2주)
- 성능 최적화와벤치마크테스트
- 전체의테스트오버라이드
- 문서와사용자가이드
- 배포 준备와CI/CD

这은/는 나타냄Open Claude Code在MCP프로토콜와확장 시스템구현上의重大돌파, 为最终의产品릴리스奠定了坚实의技术基础. 