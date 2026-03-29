using System;
using System.Collections.Generic;
using OpenTK.Graphics.OpenGL4;
using OpenTK.Mathematics;
using OpenTK.Windowing.Common;
using OpenTK.Windowing.Desktop;
using OpenTK.Windowing.GraphicsLibraryFramework;

public class SimulationWindow : GameWindow
{
    private int _vertexArrayObject;
    private int _vertexBufferObject;
    private int _shaderProgram;
    private int _positionUniformLocation;
    private int _colorUniformLocation;

    private float[] _botVertices = new float[]
    {
        -0.5f, -0.5f,  0.5f, -0.5f,  0.5f,  0.5f, -0.5f,  0.5f
    };

    private List<Bot> _population;
    private int _generation = 0;
    private int _frameCounter = 0;
    private const int FramesPerGeneration = 400;
    private const int PopulationSize = 30;
    private const double TargetPos = 8.0;
    private const double Boundary = 12.0;
    private Bot _bestBotOfGen;
    private double _bestScore = -1;

    public SimulationWindow(GameWindowSettings gameWindowSettings, NativeWindowSettings nativeWindowSettings)
        : base(gameWindowSettings, nativeWindowSettings)
    {
    }

    protected override void OnLoad()
    {
        base.OnLoad();
        GL.ClearColor(0.15f, 0.15f, 0.15f, 1.0f);
        
        _vertexArrayObject = GL.GenVertexArray();
        GL.BindVertexArray(_vertexArrayObject);

        _vertexBufferObject = GL.GenBuffer();
        GL.BindBuffer(BufferTarget.ArrayBuffer, _vertexBufferObject);
        GL.BufferData(BufferTarget.ArrayBuffer, _botVertices.Length * sizeof(float), _botVertices, BufferUsageHint.StaticDraw);

        GL.VertexAttribPointer(0, 2, VertexAttribPointerType.Float, false, 8, 0);
        GL.EnableVertexAttribArray(0);
        InitShaders();
        
        StartNewGeneration(null);
        Console.WriteLine("=== СИМУЛЯЦИЯ ЗАПУЩЕНА ===");
    }

    private void InitShaders()
    {
        string vertexSource = @"#version 330 core
layout(location = 0) in vec2 aPosition;
uniform vec2 uOffset;
void main() {
    gl_Position = vec4(aPosition + uOffset, 0.0, 1.0);
}";

        string fragmentSource = @"#version 330 core
out vec4 FragColor;
uniform vec3 uColor;
void main() {
    FragColor = vec4(uColor, 1.0);
}";

        int vertexShader = GL.CreateShader(ShaderType.VertexShader);
        GL.ShaderSource(vertexShader, vertexSource);
        GL.CompileShader(vertexShader);

        int fragmentShader = GL.CreateShader(ShaderType.FragmentShader);
        GL.ShaderSource(fragmentShader, fragmentSource);
        GL.CompileShader(fragmentShader);

        _shaderProgram = GL.CreateProgram();
        GL.AttachShader(_shaderProgram, vertexShader);
        GL.AttachShader(_shaderProgram, fragmentShader);
        GL.LinkProgram(_shaderProgram);
        GL.UseProgram(_shaderProgram);

        _positionUniformLocation = GL.GetUniformLocation(_shaderProgram, "uOffset");
        _colorUniformLocation = GL.GetUniformLocation(_shaderProgram, "uColor");

        GL.DetachShader(_shaderProgram, vertexShader);
        GL.DetachShader(_shaderProgram, fragmentShader);
        GL.DeleteShader(vertexShader);
        GL.DeleteShader(fragmentShader);
    }

    private void StartNewGeneration(Brain bestBrain)
    {
        _population = new List<Bot>();
        _bestScore = -1;        _bestBotOfGen = null;
        _frameCounter = 0;
        _generation++;

        for (int i = 0; i < PopulationSize; i++)
        {
            Bot bot;
            if (bestBrain != null)
            {
                bot = new Bot(bestBrain);
                bot.hisBrain.Mutate(0.1);
            }
            else
            {
                bot = new Bot();
            }
            bot.StartY = (i % 10) * 1.2f - 6.0f; 
            _population.Add(bot);
        }
        Console.WriteLine($"--- Поколение {_generation} ---");
    }

    protected override void OnUpdateFrame(FrameEventArgs e)
    {
        base.OnUpdateFrame(e);
        if (KeyboardState.IsKeyDown(Keys.Escape)) Close();

        foreach (var bot in _population)
        {
            if (_frameCounter == 0) bot.pos = -10.0;
            
            bot.Move();

            if (bot.pos > Boundary) bot.pos = Boundary;
            if (bot.pos < -Boundary) bot.pos = -Boundary;

            double distance = Math.Abs(TargetPos - bot.pos);
            bot.profit = 100.0 / (distance + 1.0);
            
            if (bot.pos < -5.0) bot.profit *= 0.5;

            if (bot.profit > _bestScore)
            {
                _bestScore = bot.profit;
                _bestBotOfGen = bot;
            }
        }

        _frameCounter++;
        if (_frameCounter >= FramesPerGeneration)
        {
            Console.WriteLine($"Лучший: Очки={_bestScore:F2}, Поз={_bestBotOfGen.pos:F2}");
            StartNewGeneration(_bestBotOfGen.hisBrain);
        }
    }

    protected override void OnRenderFrame(FrameEventArgs e)
    {
        base.OnRenderFrame(e);
        GL.Clear(ClearBufferMask.ColorBufferBit);
        GL.BindVertexArray(_vertexArrayObject);
        GL.UseProgram(_shaderProgram);

        foreach (var bot in _population)
        {
            GL.Uniform2(_positionUniformLocation, (float)bot.pos, (float)bot.StartY);

            if (bot == _bestBotOfGen)
                GL.Uniform3(_colorUniformLocation, 0.0f, 1.0f, 0.0f); 
            else
                GL.Uniform3(_colorUniformLocation, 1.0f, 1.0f, 1.0f); 

            GL.DrawArrays(PrimitiveType.TriangleFan, 0, 4);
        }
        SwapBuffers();
    }
}
